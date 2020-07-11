---
title: "mount"
date: 2020-03-06T20:38:56+08:00
draft: true
categories: ["技术"]
tags: ["buildah"]
---
古今陵谷茫茫。市朝往往耕桑。此地居然形胜，似曾小小兴亡。——辛弃疾《清平乐》
<!--more-->
# 文件路径
{{< highlight go "linenos=inline" >}}
github.com/containers/buildah/mount.go
{{< /highlight >}}


# Builder.Mount
{{< highlight go "linenos=inline" >}}
// Mount mounts a container's root filesystem in a location which can be
// accessed from the host, and returns the location.
func (b *Builder) Mount(label string) (string, error) {
	mountpoint, err := b.store.Mount(b.ContainerID, label)
	if err != nil {
		return "", errors.Wrapf(err, "error mounting build container %q", b.ContainerID)
	}
	b.MountPoint = mountpoint

	err = b.Save()
	if err != nil {
		return "", errors.Wrapf(err, "error saving updated state for build container %q", b.ContainerID)
	}
	return mountpoint, nil
}
{{< /highlight >}}
