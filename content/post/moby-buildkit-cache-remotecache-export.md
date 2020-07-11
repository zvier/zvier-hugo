---
title: "moby buildkit cache remotecache export"
date: 2020-03-05T22:12:40+08:00
draft: true
categories: ["技术"]
tags: ["buildkit"]
---
不是爱风尘，似被前缘误。花落花开自有时。总赖东君主。去也终须去，住也如何住！若得山花插满头，莫问奴归处。——严蕊《卜算子》
<!--more-->
# 文件路径
{{< highlight go "linenos=inline" >}}
github.com/moby/buildkit/cache/remotecache/export.go
{{< /highlight >}}

# ResolveCacheExporterFunc
{{< highlight go "linenos=inline" >}}
type ResolveCacheExporterFunc func(ctx context.Context, attrs map[string]string) (Exporter, error)
{{< /highlight >}}

# oneOffProgress
{{< highlight go "linenos=inline" >}}
func oneOffProgress(ctx context.Context, id string) func(err error) error {
    // "github.com/moby/buildkit/util/progress"
	pw, _, _ := progress.FromContext(ctx)
	now := time.Now()
	st := progress.Status{
		Started: &now,
	}
	pw.Write(id, st)
	return func(err error) error {
		now := time.Now()
		st.Completed = &now
		pw.Write(id, st)
		pw.Close()
		return err
	}
}
{{< /highlight >}}

# Exporter
{{< highlight go "linenos=inline" >}}
type Exporter interface {
	solver.CacheExporterTarget
	// Finalize finalizes and return metadata that are returned to the client
	// e.g. ExporterResponseManifestDesc
	Finalize(ctx context.Context) (map[string]string, error)
}
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
const (
	// ExportResponseManifestDesc is a key for the map returned from Exporter.Finalize.
	// The map value is a JSON string of an OCI desciptor of a manifest.
	ExporterResponseManifestDesc = "cache.manifest"
)

type contentCacheExporter struct {
	solver.CacheExporterTarget
	chains   *v1.CacheChains
	ingester content.Ingester
}

func NewExporter(ingester content.Ingester) Exporter {
	cc := v1.NewCacheChains()
	return &contentCacheExporter{CacheExporterTarget: cc, chains: cc, ingester: ingester}
}

func (ce *contentCacheExporter) Finalize(ctx context.Context) (map[string]string, error) {
	return export(ctx, ce.ingester, ce.chains)
}

func export(ctx context.Context, ingester content.Ingester, cc *v1.CacheChains) (map[string]string, error) {
	res := make(map[string]string)
	config, descs, err := cc.Marshal()
	if err != nil {
		return nil, err
	}

	// own type because oci type can't be pushed and docker type doesn't have annotations
	type manifestList struct {
		specs.Versioned

		MediaType string `json:"mediaType,omitempty"`

		// Manifests references platform specific manifests.
		Manifests []ocispec.Descriptor `json:"manifests"`
	}

	var mfst manifestList
	mfst.SchemaVersion = 2
	mfst.MediaType = images.MediaTypeDockerSchema2ManifestList

	for _, l := range config.Layers {
		dgstPair, ok := descs[l.Blob]
		if !ok {
			return nil, errors.Errorf("missing blob %s", l.Blob)
		}
		layerDone := oneOffProgress(ctx, fmt.Sprintf("writing layer %s", l.Blob))
		if err := contentutil.Copy(ctx, ingester, dgstPair.Provider, dgstPair.Descriptor); err != nil {
			return nil, layerDone(errors.Wrap(err, "error writing layer blob"))
		}
		layerDone(nil)
		mfst.Manifests = append(mfst.Manifests, dgstPair.Descriptor)
	}

	dt, err := json.Marshal(config)
	if err != nil {
		return nil, err
	}
	dgst := digest.FromBytes(dt)
	desc := ocispec.Descriptor{
		Digest:    dgst,
		Size:      int64(len(dt)),
		MediaType: v1.CacheConfigMediaTypeV0,
	}
	configDone := oneOffProgress(ctx, fmt.Sprintf("writing config %s", dgst))
	if err := content.WriteBlob(ctx, ingester, dgst.String(), bytes.NewReader(dt), desc); err != nil {
		return nil, configDone(errors.Wrap(err, "error writing config blob"))
	}
	configDone(nil)

	mfst.Manifests = append(mfst.Manifests, desc)

	dt, err = json.Marshal(mfst)
	if err != nil {
		return nil, errors.Wrap(err, "failed to marshal manifest")
	}
	dgst = digest.FromBytes(dt)

	desc = ocispec.Descriptor{
		Digest:    dgst,
		Size:      int64(len(dt)),
		MediaType: mfst.MediaType,
	}
	mfstDone := oneOffProgress(ctx, fmt.Sprintf("writing manifest %s", dgst))
	if err := content.WriteBlob(ctx, ingester, dgst.String(), bytes.NewReader(dt), desc); err != nil {
		return nil, mfstDone(errors.Wrap(err, "error writing manifest blob"))
	}
	descJSON, err := json.Marshal(desc)
	if err != nil {
		return nil, err
	}
	res[ExporterResponseManifestDesc] = string(descJSON)
	mfstDone(nil)
	return res, nil
}
{{< /highlight >}}

