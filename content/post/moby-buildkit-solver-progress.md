---
title: "moby buildkit solver progress"
date: 2020-03-01T21:48:29+08:00
draft: true
categories: ["技术"]
tags: ["buildkit"]
---
庭院深深深几许，杨柳堆烟，帘幕无重数。玉勒雕鞍游冶处。楼高不见章台路。雨横风狂三月暮，门掩黄昏，无计留春住，泪眼问花花不语，乱红飞过秋千去。——欧阳修《蝶恋花》
<!--more-->
# 文件路径
{{< highlight go "linenos=inline" >}}
github.com/moby/buildkit/solver/progress.go
{{< /highlight >}}

# Job.Status
{{< highlight go "linenos=inline" >}}
func (j *Job) Status(ctx context.Context, ch chan *client.SolveStatus) error {
	vs := &vertexStream{cache: map[digest.Digest]*client.Vertex{}}
	pr := j.pr.Reader(ctx)
	defer func() {
		if enc := vs.encore(); len(enc) > 0 {
			ch <- &client.SolveStatus{Vertexes: enc}
		}
		close(ch)
	}()

	for {
		p, err := pr.Read(ctx)
		if err != nil {
			if err == io.EOF {
				return nil
			}
			return err
		}
		ss := &client.SolveStatus{}
		for _, p := range p {
			switch v := p.Sys.(type) {
			case client.Vertex:
				ss.Vertexes = append(ss.Vertexes, vs.append(v)...)

			case progress.Status:
				vtx, ok := p.Meta("vertex")
				if !ok {
					logrus.Warnf("progress %s status without vertex info", p.ID)
					continue
				}
				vs := &client.VertexStatus{
					ID:        p.ID,
					Vertex:    vtx.(digest.Digest),
					Name:      v.Action,
					Total:     int64(v.Total),
					Current:   int64(v.Current),
					Timestamp: p.Timestamp,
					Started:   v.Started,
					Completed: v.Completed,
				}
				ss.Statuses = append(ss.Statuses, vs)
			case client.VertexLog:
				vtx, ok := p.Meta("vertex")
				if !ok {
					logrus.Warnf("progress %s log without vertex info", p.ID)
					continue
				}
				v.Vertex = vtx.(digest.Digest)
				v.Timestamp = p.Timestamp
				ss.Logs = append(ss.Logs, &v)
			}
		}
		select {
		case <-ctx.Done():
			return ctx.Err()
		case ch <- ss:
		}
	}
}
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
type vertexStream struct {
	cache map[digest.Digest]*client.Vertex
}

func (vs *vertexStream) append(v client.Vertex) []*client.Vertex {
	var out []*client.Vertex
	vs.cache[v.Digest] = &v
	if v.Started != nil {
		for _, inp := range v.Inputs {
			if inpv, ok := vs.cache[inp]; ok {
				if !inpv.Cached && inpv.Completed == nil {
					inpv.Cached = true
					inpv.Started = v.Started
					inpv.Completed = v.Started
					out = append(out, vs.append(*inpv)...)
					delete(vs.cache, inp)
				}
			}
		}
	}
	vcopy := v
	return append(out, &vcopy)
}

func (vs *vertexStream) encore() []*client.Vertex {
	var out []*client.Vertex
	for _, v := range vs.cache {
		if v.Started != nil && v.Completed == nil {
			now := time.Now()
			v.Completed = &now
			v.Error = context.Canceled.Error()
			out = append(out, v)
		}
	}
	return out
}
{{< /highlight >}}


{{< highlight go "linenos=inline" >}}
{{< /highlight >}}

