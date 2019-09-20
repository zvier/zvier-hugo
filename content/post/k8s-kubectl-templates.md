---
title: "k8s templates"
date: 2019-09-07T10:33:01+08:00
draft: true
categories: [""]
tags: ["k8s"]
---

# CommandGroup
{{< highlight go "linenos=inline" >}}
// k8s.io/kubernetes/pkg/kubectl/util/templates/command_groups.go
type CommandGroup struct {
    Message  string
    Commands []*cobra.Command
}

type CommandGroups []CommandGroup
{{< /highlight >}}

