---
title: "docker-docker-api-server-backend-build-backend"
date: 2019-10-11T22:14:53+08:00
draft: true
categories: ["技术"]
tags: ["docker"]
---

# 简述
雨中黄叶树，灯下白头人。——司空曙《喜外弟卢纶见宿》

<!--more-->
# 文件路径
{{< highlight go "linenos=inline" >}}
github.com/docker/docker/api/server/backend/build/backend.go
{{< /highlight >}}

import (
	"github.com/docker/docker/api/types"
	"github.com/docker/docker/builder"
	buildkit "github.com/docker/docker/builder/builder-next"
	"github.com/docker/docker/builder/fscache"
	"github.com/docker/docker/image"
	"github.com/docker/docker/pkg/stringid"
	"github.com/pkg/errors"
	"golang.org/x/sync/errgroup"
	"google.golang.org/grpc"
)

# ImageComponent
{{< highlight go "linenos=inline" >}}
import	"github.com/docker/distribution/reference"
{{< /highlight >}}

ImageComponent定义了一组镜像相关的操作接口   
{{< highlight go "linenos=inline" >}}
// ImageComponent provides an interface for working with images
type ImageComponent interface {
	SquashImage(from string, to string) (string, error)
	TagImageWithReference(image.ID, reference.Named) error
}
{{< /highlight >}}

# Builder
Builder也是一个interface，定义了build相关的操作方法  
{{< highlight go "linenos=inline" >}}
import	"github.com/docker/docker/api/types/backend"
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
// Builder defines interface for running a build  
type Builder interface {
	Build(context.Context, backend.BuildConfig) (*builder.Result, error)
}
{{< /highlight >}}

# Backend
{{< highlight go "linenos=inline" >}}
// Backend provides build functionality to the API router
type Backend struct {
	builder        Builder
	fsCache        *fscache.FSCache
	imageComponent ImageComponent
	buildkit       *buildkit.Builder
}
{{< /highlight >}}

# NewBackend
{{< highlight go "linenos=inline" >}}
// NewBackend creates a new build backend from components
func NewBackend(components ImageComponent, builder Builder, fsCache *fscache.FSCache, buildkit *buildkit.Builder) (*Backend, error) {
	return &Backend{imageComponent: components, builder: builder, fsCache: fsCache, buildkit: buildkit}, nil
}
{{< /highlight >}}

# RegisterGRPC
{{< highlight go "linenos=inline" >}}
// RegisterGRPC registers buildkit controller to the grpc server.
func (b *Backend) RegisterGRPC(s *grpc.Server) {
	if b.buildkit != nil {
		b.buildkit.RegisterGRPC(s)
	}
}
{{< /highlight >}}

# Build
{{< highlight go "linenos=inline" >}}
// Build builds an image from a Source
func (b *Backend) Build(ctx context.Context, config backend.BuildConfig) (string, error) {
	options := config.Options
	useBuildKit := options.Version == types.BuilderBuildKit

	tagger, err := NewTagger(b.imageComponent, config.ProgressWriter.StdoutFormatter, options.Tags)
	if err != nil {
		return "", err
	}

	var build *builder.Result
	if useBuildKit {
		build, err = b.buildkit.Build(ctx, config)
		if err != nil {
			return "", err
		}
	} else {
		build, err = b.builder.Build(ctx, config)
		if err != nil {
			return "", err
		}
	}

	if build == nil {
		return "", nil
	}

	var imageID = build.ImageID
	if options.Squash {
		if imageID, err = squashBuild(build, b.imageComponent); err != nil {
			return "", err
		}
		if config.ProgressWriter.AuxFormatter != nil {
			if err = config.ProgressWriter.AuxFormatter.Emit("moby.image.id", types.BuildResult{ID: imageID}); err != nil {
				return "", err
			}
		}
	}

	if !useBuildKit {
		stdout := config.ProgressWriter.StdoutFormatter
		fmt.Fprintf(stdout, "Successfully built %s\n", stringid.TruncateID(imageID))
	}
	if imageID != "" {
		err = tagger.TagImages(image.ID(imageID))
	}
	return imageID, err
}
{{< /highlight >}}

# Backend.PruneCache
{{< highlight go "linenos=inline" >}}
// PruneCache removes all cached build sources
func (b *Backend) PruneCache(ctx context.Context, opts types.BuildCachePruneOptions) (*types.BuildCachePruneReport, error) {
	eg, ctx := errgroup.WithContext(ctx)

	var fsCacheSize uint64
	eg.Go(func() error {
		var err error
		fsCacheSize, err = b.fsCache.Prune(ctx)
		if err != nil {
			return errors.Wrap(err, "failed to prune fscache")
		}
		return nil
	})

	var buildCacheSize int64
	var cacheIDs []string
	eg.Go(func() error {
		var err error
		buildCacheSize, cacheIDs, err = b.buildkit.Prune(ctx, opts)
		if err != nil {
			return errors.Wrap(err, "failed to prune build cache")
		}
		return nil
	})

	if err := eg.Wait(); err != nil {
		return nil, err
	}

	return &types.BuildCachePruneReport{SpaceReclaimed: fsCacheSize + uint64(buildCacheSize), CachesDeleted: cacheIDs}, nil
}
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
// Cancel cancels the build by ID
func (b *Backend) Cancel(ctx context.Context, id string) error {
	return b.buildkit.Cancel(ctx, id)
}
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
func squashBuild(build *builder.Result, imageComponent ImageComponent) (string, error) {
	var fromID string
	if build.FromImage != nil {
		fromID = build.FromImage.ImageID()
	}
	imageID, err := imageComponent.SquashImage(build.ImageID, fromID)
	if err != nil {
		return "", errors.Wrap(err, "error squashing image")
	}
	return imageID, nil
}

{{< /highlight >}}
