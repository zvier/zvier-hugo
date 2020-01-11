---
title: "ctr images pull"
date: 2019-12-29T10:37:38+08:00
draft: true
categories: ["技术"]
tags: ["containerd"]
---
明日萧条醉尽醒，残花烂熳开何益。——杜甫
<!--more-->
# 简述
<code>ctr images pull</code>命令下载容器镜像，以下以<code>ctr images pull docker.io/library/hello-world:latest</code>为例，分析整个下载流程
# ctr pull客户端
{{< highlight go "linenos=inline" >}}
// github.com/containerd/containerd/cmd/ctr/commands/images
package images

import (
	"fmt"

	"github.com/containerd/containerd"
	"github.com/containerd/containerd/cmd/ctr/commands"
	"github.com/containerd/containerd/cmd/ctr/commands/content"
	"github.com/containerd/containerd/images"
	"github.com/containerd/containerd/log"
	"github.com/containerd/containerd/platforms"
	ocispec "github.com/opencontainers/image-spec/specs-go/v1"
	"github.com/pkg/errors"
	"github.com/urfave/cli"
)

var pullCommand = cli.Command{
	Name:      "pull",
	Usage:     "pull an image from a remote",
	ArgsUsage: "[flags] <ref>",
	Description: `Fetch and prepare an image for use in containerd.

After pulling an image, it should be ready to use the same reference in a run
command. As part of this process, we do the following:

1. Fetch all resources into containerd.
2. Prepare the snapshot filesystem with the pulled resources.
3. Register metadata for the image.
`,
	Flags: append(append(commands.RegistryFlags, append(commands.SnapshotterFlags, commands.LabelFlag)...),
		cli.StringSliceFlag{
			Name:  "platform",
			Usage: "Pull content from a specific platform",
			Value: &cli.StringSlice{},
		},
		cli.BoolFlag{
			Name:  "all-platforms",
			Usage: "pull content and metadata from all platforms",
		},
		cli.BoolFlag{
			Name:  "all-metadata",
			Usage: "Pull metadata for all platforms",
		},
	),
	Action: func(context *cli.Context) error {
		var (
			ref = context.Args().First()
		)
		if ref == "" {
			return fmt.Errorf("please provide an image reference to pull")
		}

		client, ctx, cancel, err := commands.NewClient(context)
		if err != nil {
			return err
		}
		defer cancel()

		ctx, done, err := client.WithLease(ctx)
		if err != nil {
			return err
		}
		defer done(ctx)

		config, err := content.NewFetchConfig(ctx, context)
		if err != nil {
			return err
		}

		img, err := content.Fetch(ctx, client, ref, config)
		if err != nil {
			return err
		}

		log.G(ctx).WithField("image", ref).Debug("unpacking")

		// TODO: Show unpack status

		var p []ocispec.Platform
		if context.Bool("all-platforms") {
			p, err = images.Platforms(ctx, client.ContentStore(), img.Target)
			if err != nil {
				return errors.Wrap(err, "unable to resolve image platforms")
			}
		} else {
			for _, s := range context.StringSlice("platform") {
				ps, err := platforms.Parse(s)
				if err != nil {
					return errors.Wrapf(err, "unable to parse platform %s", s)
				}
				p = append(p, ps)
			}
		}
		if len(p) == 0 {
			p = append(p, platforms.DefaultSpec())
		}

		for _, platform := range p {
			fmt.Printf("unpacking %s %s...\n", platforms.Format(platform), img.Target.Digest)
			i := containerd.NewImageWithPlatform(client, img, platforms.Only(platform))
			err = i.Unpack(ctx, context.String("snapshotter"))
			if err != nil {
				return err
			}
		}

		fmt.Println("done")
		return nil
	},
}
{{< /highlight >}}

# ctr content客户端
{{< highlight go "linenos=inline" >}}
// github.com/containerd/containerd/cmd/ctr/commands/content/fetch.go
// Fetch loads all resources into the content store and returns the image
func Fetch(ctx context.Context, client *containerd.Client, ref string, config *FetchConfig) (images.Image, error) {
    ongoing := newJobs(ref)

    pctx, stopProgress := context.WithCancel(ctx)
    progress := make(chan struct{})

    go func() {
        if config.ProgressOutput != nil {
            // no progress bar, because it hides some debug logs
            showProgress(pctx, ongoing, client.ContentStore(), config.ProgressOutput)
        }
        close(progress)
    }()

    h := images.HandlerFunc(func(ctx context.Context, desc ocispec.Descriptor) ([]ocispec.Descriptor, error) {
        if desc.MediaType != images.MediaTypeDockerSchema1Manifest {
            ongoing.add(desc)
        }
        return nil, nil
    })

    log.G(pctx).WithField("image", ref).Debug("fetching")
    labels := commands.LabelArgs(config.Labels)
    opts := []containerd.RemoteOpt{
        containerd.WithPullLabels(labels),
        containerd.WithResolver(config.Resolver),
        containerd.WithImageHandler(h),
        containerd.WithSchema1Conversion,
    }

    if config.AllMetadata {
        opts = append(opts, containerd.WithAllMetadata())
    }
    if config.PlatformMatcher != nil {
        opts = append(opts, containerd.WithPlatformMatcher(config.PlatformMatcher))
    } else {
        for _, platform := range config.Platforms {
            opts = append(opts, containerd.WithPlatform(platform))
        }
    }

    img, err := client.Fetch(pctx, ref, opts...)
    stopProgress()
    if err != nil {
        return images.Image{}, err
    }

    <-progress
    return img, nil
}
{{< /highlight >}}

# Client.Fetch
{{< highlight go "linenos=inline" >}}
// Fetch downloads the provided content into containerd's content store
// and returns a non-platform specific image reference
func (c *Client) Fetch(ctx context.Context, ref string, opts ...RemoteOpt) (images.Image, error) {
    fetchCtx := defaultRemoteContext()
    for _, o := range opts {
        if err := o(c, fetchCtx); err != nil {
            return images.Image{}, err
        }
    }

    if fetchCtx.Unpack {
        return images.Image{}, errors.Wrap(errdefs.ErrNotImplemented, "unpack on fetch not supported, try pull")
    }

    if fetchCtx.PlatformMatcher == nil {
        if len(fetchCtx.Platforms) == 0 {
            fetchCtx.PlatformMatcher = platforms.All
        } else {
            var ps []ocispec.Platform
            for _, s := range fetchCtx.Platforms {
                p, err := platforms.Parse(s)
                if err != nil {
                    return images.Image{}, errors.Wrapf(err, "invalid platform %s", s)
                }
                ps = append(ps, p)
            }

            fetchCtx.PlatformMatcher = platforms.Any(ps...)
        }
    }

    ctx, done, err := c.WithLease(ctx)
    if err != nil {
        return images.Image{}, err
    }
    defer done(ctx)

    return c.fetch(ctx, fetchCtx, ref, 0)
}
{{< /highlight >}}

# client.fetch
{{< highlight go "linenos=inline" >}}
// github.com/containerd/containerd/pull.go
func (c *Client) fetch(ctx context.Context, rCtx *RemoteContext, ref string, limit int) (images.Image, error) {
	store := c.ContentStore()
	name, desc, err := rCtx.Resolver.Resolve(ctx, ref)
	if err != nil {
		return images.Image{}, errors.Wrapf(err, "failed to resolve reference %q", ref)
	}

	fetcher, err := rCtx.Resolver.Fetcher(ctx, name)
	if err != nil {
		return images.Image{}, errors.Wrapf(err, "failed to get fetcher for %q", name)
	}

	var (
		handler images.Handler

		isConvertible bool
		converterFunc func(context.Context, ocispec.Descriptor) (ocispec.Descriptor, error)
		limiter       *semaphore.Weighted
	)

	if desc.MediaType == images.MediaTypeDockerSchema1Manifest && rCtx.ConvertSchema1 {
		schema1Converter := schema1.NewConverter(store, fetcher)

		handler = images.Handlers(append(rCtx.BaseHandlers, schema1Converter)...)

		isConvertible = true

		converterFunc = func(ctx context.Context, _ ocispec.Descriptor) (ocispec.Descriptor, error) {
			return schema1Converter.Convert(ctx)
		}
	} else {
		// Get all the children for a descriptor
		childrenHandler := images.ChildrenHandler(store)
		// Set any children labels for that content
		childrenHandler = images.SetChildrenLabels(store, childrenHandler)
		if rCtx.AllMetadata {
			// Filter manifests by platforms but allow to handle manifest
			// and configuration for not-target platforms
			childrenHandler = remotes.FilterManifestByPlatformHandler(childrenHandler, rCtx.PlatformMatcher)
		} else {
			// Filter children by platforms if specified.
			childrenHandler = images.FilterPlatforms(childrenHandler, rCtx.PlatformMatcher)
		}
		// Sort and limit manifests if a finite number is needed
		if limit > 0 {
			childrenHandler = images.LimitManifests(childrenHandler, rCtx.PlatformMatcher, limit)
		}

		// set isConvertible to true if there is application/octet-stream media type
		convertibleHandler := images.HandlerFunc(
			func(_ context.Context, desc ocispec.Descriptor) ([]ocispec.Descriptor, error) {
				if desc.MediaType == docker.LegacyConfigMediaType {
					isConvertible = true
				}

				return []ocispec.Descriptor{}, nil
			},
		)

		appendDistSrcLabelHandler, err := docker.AppendDistributionSourceLabel(store, ref)
		if err != nil {
			return images.Image{}, err
		}

		handlers := append(rCtx.BaseHandlers,
			remotes.FetchHandler(store, fetcher),
			convertibleHandler,
			childrenHandler,
			appendDistSrcLabelHandler,
		)

		handler = images.Handlers(handlers...)

		converterFunc = func(ctx context.Context, desc ocispec.Descriptor) (ocispec.Descriptor, error) {
			return docker.ConvertManifest(ctx, store, desc)
		}
	}

	if rCtx.HandlerWrapper != nil {
		handler = rCtx.HandlerWrapper(handler)
	}

	if rCtx.MaxConcurrentDownloads > 0 {
		limiter = semaphore.NewWeighted(int64(rCtx.MaxConcurrentDownloads))
	}

	if err := images.Dispatch(ctx, handler, limiter, desc); err != nil {
		return images.Image{}, err
	}

	if isConvertible {
		if desc, err = converterFunc(ctx, desc); err != nil {
			return images.Image{}, err
		}
	}

	img := images.Image{
		Name:   name,
		Target: desc,
		Labels: rCtx.Labels,
	}

	is := c.ImageService()
	for {
		if created, err := is.Create(ctx, img); err != nil {
			if !errdefs.IsAlreadyExists(err) {
				return images.Image{}, err
			}

			updated, err := is.Update(ctx, img)
			if err != nil {
				// if image was removed, try create again
				if errdefs.IsNotFound(err) {
					continue
				}
				return images.Image{}, err
			}

			img = updated
		} else {
			img = created
		}

		return img, nil
	}
}
{{< /highlight >}}

# RemoteContext

{{< highlight go "linenos=inline" >}}
// RemoteContext is used to configure object resolutions and transfers with
// remote content stores and image providers.
type RemoteContext struct {
    // Resolver is used to resolve names to objects, fetchers, and pushers.
    // If no resolver is provided, defaults to Docker registry resolver.
    Resolver remotes.Resolver

    // PlatformMatcher is used to match the platforms for an image
    // operation and define the preference when a single match is required
    // from multiple platforms.
    PlatformMatcher platforms.MatchComparer

    // Unpack is done after an image is pulled to extract into a snapshotter.
    // It is done simultaneously for schema 2 images when they are pulled.
    // If an image is not unpacked on pull, it can be unpacked any time
    // afterwards. Unpacking is required to run an image.
    Unpack bool

    // UnpackOpts handles options to the unpack call.
    UnpackOpts []UnpackOpt

    // Snapshotter used for unpacking
    Snapshotter string

    // Labels to be applied to the created image
    Labels map[string]string

    // BaseHandlers are a set of handlers which get are called on dispatch.
    // These handlers always get called before any operation specific
    // handlers.
    BaseHandlers []images.Handler
    // HandlerWrapper wraps the handler which gets sent to dispatch.
    // Unlike BaseHandlers, this can run before and after the built
    // in handlers, allowing operations to run on the descriptor
    // after it has completed transferring.
    HandlerWrapper func(images.Handler) images.Handler

    // ConvertSchema1 is whether to convert Docker registry schema 1
    // manifests. If this option is false then any image which resolves
    // to schema 1 will return an error since schema 1 is not supported.
    ConvertSchema1 bool

    // Platforms defines which platforms to handle when doing the image operation.
    // Platforms is ignored when a PlatformMatcher is set, otherwise the
    // platforms will be used to create a PlatformMatcher with no ordering
    // preference.
    Platforms []string

    // MaxConcurrentDownloads is the max concurrent content downloads for each pull.
    MaxConcurrentDownloads int

    // AllMetadata downloads all manifests and known-configuration files
    AllMetadata bool
}
{{< /highlight >}}

# Client.ContentStore
Client.ContentStore返回client的content store对象，如果没有，则创建并返回一个proxy.proxyContentStore对象。
{{< highlight go "linenos=inline" >}}
// github.com/containerd/containerd/client.go
// ContentStore returns the underlying content Store
func (c *Client) ContentStore() content.Store {
    if c.contentStore != nil {
        return c.contentStore
    }
    c.connMu.Lock()
    defer c.connMu.Unlock()
    return contentproxy.NewContentStore(contentapi.NewContentClient(c.conn))
}
{{< /highlight >}}

# NewContentStore
NewContentStore根据GRPC ContentClient，返回一个proxyContentStore对象，它实现了content.Store接口
{{< highlight go "linenos=inline" >}}
// github.com/containerd/containerd/content/proxy/content_store.go
package proxy

import (
    "context"
    "io"

    contentapi "github.com/containerd/containerd/api/services/content/v1"
    "github.com/containerd/containerd/content"
    "github.com/containerd/containerd/errdefs"
    protobuftypes "github.com/gogo/protobuf/types"
    digest "github.com/opencontainers/go-digest"
    ocispec "github.com/opencontainers/image-spec/specs-go/v1"
)

type proxyContentStore struct {
    client contentapi.ContentClient
}

// NewContentStore returns a new content store which communicates over a GRPC
// connection using the containerd content GRPC API.
func NewContentStore(client contentapi.ContentClient) content.Store {
    return &proxyContentStore{
        client: client,
    }
}
{{< /highlight >}}
