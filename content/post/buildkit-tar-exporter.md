---
title: "buildkit tar exporter"
date: 2020-02-10T16:15:57+08:00
draft: true
categories: ["技术"]
tags: ["buildkit"]
---
燕子不归春事晚，一汀烟雨杏花寒。——唐.戴叔伦《苏溪亭》
<!--more-->
# 包
{{< highlight go "linenos=inline" >}}
// github.com/moby/buildkit/exporter/tar/export.go
package local

import (
	"context"
	"io/ioutil"
	"os"
	"strings"
	"time"

	"github.com/docker/docker/pkg/idtools"
	"github.com/moby/buildkit/cache"
	"github.com/moby/buildkit/exporter"
	"github.com/moby/buildkit/session"
	"github.com/moby/buildkit/session/filesync"
	"github.com/moby/buildkit/snapshot"
	"github.com/moby/buildkit/util/progress"
	"github.com/pkg/errors"
	"github.com/tonistiigi/fsutil"
	fstypes "github.com/tonistiigi/fsutil/types"
)
{{< /highlight >}}

# Opt
{{< highlight go "linenos=inline" >}}
type Opt struct {
	SessionManager *session.Manager
}
{{< /highlight >}}

# localExporter
{{< highlight go "linenos=inline" >}}
type localExporter struct {
	opt Opt
	// session manager
}
{{< /highlight >}}

# New
{{< highlight go "linenos=inline" >}}
func New(opt Opt) (exporter.Exporter, error) {
	le := &localExporter{opt: opt}
	return le, nil
}
{{< /highlight >}}

# localExporter.Resolve
{{< highlight go "linenos=inline" >}}
func (e *localExporter) Resolve(ctx context.Context, opt map[string]string) (exporter.ExporterInstance, error) {
	id := session.FromContext(ctx)
	if id == "" {
		return nil, errors.New("could not access local files without session")
	}

	timeoutCtx, cancel := context.WithTimeout(ctx, 5*time.Second)
	defer cancel()

	caller, err := e.opt.SessionManager.Get(timeoutCtx, id)
	if err != nil {
		return nil, err
	}

	li := &localExporterInstance{localExporter: e, caller: caller}
	return li, nil
}
{{< /highlight >}}

# localExporterInstance
{{< highlight go "linenos=inline" >}}
type localExporterInstance struct {
	*localExporter
	caller session.Caller
}
{{< /highlight >}}

# localExporterInstance.Name 
{{< highlight go "linenos=inline" >}}
func (e *localExporterInstance) Name() string {
	return "exporting to client"
}
{{< /highlight >}}

# localExporterInstance.Export
{{< highlight go "linenos=inline" >}}
func (e *localExporterInstance) Export(ctx context.Context, inp exporter.Source) (map[string]string, error) {
	var defers []func()

	defer func() {
		for i := len(defers) - 1; i >= 0; i-- {
			defers[i]()
		}
	}()

	getDir := func(ctx context.Context, k string, ref cache.ImmutableRef) (*fsutil.Dir, error) {
		var src string
		var err error
		var idmap *idtools.IdentityMapping
		if ref == nil {
			src, err = ioutil.TempDir("", "buildkit")
			if err != nil {
				return nil, err
			}
			defers = append(defers, func() { os.RemoveAll(src) })
		} else {
			mount, err := ref.Mount(ctx, true)
			if err != nil {
				return nil, err
			}

			lm := snapshot.LocalMounter(mount)

			src, err = lm.Mount()
			if err != nil {
				return nil, err
			}

			idmap = mount.IdentityMapping()

			defers = append(defers, func() { lm.Unmount() })
		}

		walkOpt := &fsutil.WalkOpt{}

		if idmap != nil {
			walkOpt.Map = func(p string, st *fstypes.Stat) bool {
				uid, gid, err := idmap.ToContainer(idtools.Identity{
					UID: int(st.Uid),
					GID: int(st.Gid),
				})
				if err != nil {
					return false
				}
				st.Uid = uint32(uid)
				st.Gid = uint32(gid)
				return true
			}
		}

		return &fsutil.Dir{
			FS: fsutil.NewFS(src, walkOpt),
			Stat: fstypes.Stat{
				Mode: uint32(os.ModeDir | 0755),
				Path: strings.Replace(k, "/", "_", -1),
			},
		}, nil
	}

	var fs fsutil.FS

	if len(inp.Refs) > 0 {
		dirs := make([]fsutil.Dir, 0, len(inp.Refs))
		for k, ref := range inp.Refs {
			d, err := getDir(ctx, k, ref)
			if err != nil {
				return nil, err
			}
			dirs = append(dirs, *d)
		}
		var err error
		fs, err = fsutil.SubDirFS(dirs)
		if err != nil {
			return nil, err
		}
	} else {
		d, err := getDir(ctx, "", inp.Ref)
		if err != nil {
			return nil, err
		}
		fs = d.FS
	}

	w, err := filesync.CopyFileWriter(ctx, nil, e.caller)
	if err != nil {
		return nil, err
	}
	report := oneOffProgress(ctx, "sending tarball")
	if err := fsutil.WriteTar(ctx, fs, w); err != nil {
		w.Close()
		return nil, report(err)
	}
	return nil, report(w.Close())
}
{{< /highlight >}}

# oneOffProgress
{{< highlight go "linenos=inline" >}}
func oneOffProgress(ctx context.Context, id string) func(err error) error {
	pw, _, _ := progress.FromContext(ctx)
	now := time.Now()
	st := progress.Status{
		Started: &now,
	}
	pw.Write(id, st)
	return func(err error) error {
		// TODO: set error on status
		now := time.Now()
		st.Completed = &now
		pw.Write(id, st)
		pw.Close()
		return err
	}
}
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
{{< /highlight >}}
{{< highlight go "linenos=inline" >}}
{{< /highlight >}}
