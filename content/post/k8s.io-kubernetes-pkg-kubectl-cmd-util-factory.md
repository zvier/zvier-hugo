---
title: "k8s.io-kubernetes-pkg-kubectl-cmd-util-factory"
date: 2019-09-07T18:55:01+08:00
draft: true
categories: [""]
tags: ["k8s"]
---
# 简述
<!--more-->
# 文件路径
{{< highlight go "linenos=inline" >}}
k8s.io/kubernetes/pkg/kubectl/cmd/util/factory.go
{{< /highlight >}}

# Factory
Factory提供了一种抽象，允许kubectl命令行扩展出多种资源类型和不同的API集合
{{< highlight go "linenos=inline" >}}
import (
	"k8s.io/apimachinery/pkg/api/meta"
	"k8s.io/cli-runtime/pkg/genericclioptions"
	"k8s.io/cli-runtime/pkg/resource"
	"k8s.io/client-go/dynamic"
	"k8s.io/client-go/kubernetes"
	restclient "k8s.io/client-go/rest"
	"k8s.io/kubernetes/pkg/kubectl/cmd/util/openapi"
	"k8s.io/kubernetes/pkg/kubectl/validation"
)
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
// Factory provides abstractions that allow the Kubectl command to be extended across multiple types
// of resources and different API sets.
// The rings are here for a reason. In order for composers to be able to provide alternative factory implementations
// they need to provide low level pieces of *certain* functions so that when the factory calls back into itself
// it uses the custom version of the function. Rather than try to enumerate everything that someone would want to override
// we split the factory into rings, where each ring can depend on methods in an earlier ring, but cannot depend
// upon peer methods in its own ring.
// TODO: make the functions interfaces
// TODO: pass the various interfaces on the factory directly into the command constructors (so the
// commands are decoupled from the factory).
type Factory interface {
	genericclioptions.RESTClientGetter

	// DynamicClient returns a dynamic client ready for use
	DynamicClient() (dynamic.Interface, error)

	// KubernetesClientSet gives you back an external clientset
	KubernetesClientSet() (*kubernetes.Clientset, error)

	// Returns a RESTClient for accessing Kubernetes resources or an error.
	RESTClient() (*restclient.RESTClient, error)

	// NewBuilder returns an object that assists in loading objects from both disk and the server
	// and which implements the common patterns for CLI interactions with generic resources.
	NewBuilder() *resource.Builder

	// Returns a RESTClient for working with the specified RESTMapping or an error. This is intended
	// for working with arbitrary resources and is not guaranteed to point to a Kubernetes APIServer.
	ClientForMapping(mapping *meta.RESTMapping) (resource.RESTClient, error)
	// Returns a RESTClient for working with Unstructured objects.
	UnstructuredClientForMapping(mapping *meta.RESTMapping) (resource.RESTClient, error)

	// Returns a schema that can validate objects stored on disk.
	Validator(validate bool) (validation.Schema, error)
	// OpenAPISchema returns the schema openapi schema definition
	OpenAPISchema() (openapi.Resources, error)
}
{{< /highlight >}}
