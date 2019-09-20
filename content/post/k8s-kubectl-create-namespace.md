---
title: "K8s Kubectl Create Namespace"
date: 2019-09-07T10:49:46+08:00
draft: true
categories: [""]
tags: ["k8s"]
---
# NewCmdCreateNamespace
{{< highlight go "linenos=inline" >}}
// k8s.io/kubernetes/pkg/kubectl/cmd/create/create_namespace.go
// NewCmdCreateNamespace is a macro command to create a new namespace
func NewCmdCreateNamespace(f cmdutil.Factory, ioStreams genericclioptions.IOStreams) *cobra.Command {
    options := &NamespaceOpts{
        CreateSubcommandOptions: NewCreateSubcommandOptions(ioStreams),
    }

    cmd := &cobra.Command{
        Use:                   "namespace NAME [--dry-run]",
        DisableFlagsInUseLine: true,
        Aliases:               []string{"ns"},
        Short:                 i18n.T("Create a namespace with the specified name"),
        Long:                  namespaceLong,
        Example:               namespaceExample,
        Run: func(cmd *cobra.Command, args []string) {
            cmdutil.CheckErr(options.Complete(f, cmd, args))
            cmdutil.CheckErr(options.Run())
        },
    }

    options.CreateSubcommandOptions.PrintFlags.AddFlags(cmd)

    cmdutil.AddApplyAnnotationFlags(cmd)
    cmdutil.AddValidateFlags(cmd)
    cmdutil.AddGeneratorFlags(cmd, generateversioned.NamespaceV1GeneratorName)

    return cmd
}
{{< /highlight >}}

# NamespaceOpts
{{< highlight go "linenos=inline" >}}
// NamespaceOpts is the options for 'create namespare' sub command
type NamespaceOpts struct {
    CreateSubcommandOptions *CreateSubcommandOptions
}
{{< /highlight >}}

# CreateSubcommandOptions
{{< highlight go "linenos=inline" >}}
// k8s.io/kubernetes/pkg/kubectl/cmd/create/create.go
// CreateSubcommandOptions is an options struct to support create subcommands
type CreateSubcommandOptions struct {
    // PrintFlags holds options necessary for obtaining a printer
    PrintFlags *genericclioptions.PrintFlags
    // Name of resource being created
    Name string
    // StructuredGenerator is the resource generator for the object being created
    StructuredGenerator generate.StructuredGenerator
    // DryRun is true if the command should be simulated but not run against the server
    DryRun           bool
    CreateAnnotation bool

    Namespace        string
    EnforceNamespace bool

    Mapper        meta.RESTMapper
    DynamicClient dynamic.Interface

    PrintObj printers.ResourcePrinterFunc

    genericclioptions.IOStreams
}
{{< /highlight >}}

# NewCreateSubcommandOptions(ioStreams
{{< highlight go "linenos=inline" >}}
// NewCreateSubcommandOptions returns initialized CreateSubcommandOptions
func NewCreateSubcommandOptions(ioStreams genericclioptions.IOStreams) *CreateSubcommandOptions {
    return &CreateSubcommandOptions{
        PrintFlags: genericclioptions.NewPrintFlags("created").WithTypeSetter(scheme.Scheme),
        IOStreams:  ioStreams,
    }
}
{{< /highlight >}}

# Run
{{< highlight go "linenos=inline" >}}
// Run calls the CreateSubcommandOptions.Run in NamespaceOpts instance
func (o *NamespaceOpts) Run() error {
    return o.CreateSubcommandOptions.Run()
}
{{< /highlight >}}

# CreateSubcommandOptions.Run
{{< highlight go "linenos=inline" >}}
// Run executes a create subcommand using the specified options
func (o *CreateSubcommandOptions) Run() error {
    obj, err := o.StructuredGenerator.StructuredGenerate()
    if err != nil {
        return err
    }
    if !o.DryRun {
        // create subcommands have compiled knowledge of things they create, so type them directly
        gvks, _, err := scheme.Scheme.ObjectKinds(obj)
        if err != nil {
            return err
        }
        gvk := gvks[0]
        mapping, err := o.Mapper.RESTMapping(schema.GroupKind{Group: gvk.Group, Kind: gvk.Kind}, gvk.Version)
        if err != nil {
            return err
        }

        if err := kubectl.CreateOrUpdateAnnotation(o.CreateAnnotation, obj, scheme.DefaultJSONEncoder()); err != nil {
            return err
        }

        asUnstructured := &unstructured.Unstructured{}

        if err := scheme.Scheme.Convert(obj, asUnstructured, nil); err != nil {
            return err
        }
        if mapping.Scope.Name() == meta.RESTScopeNameRoot {
            o.Namespace = ""
        }
        actualObject, err := o.DynamicClient.Resource(mapping.Resource).Namespace(o.Namespace).Create(asUnstructured, metav1.CreateOptions{})
        if err != nil {
            return err
        }

        // ensure we pass a versioned object to the printer
        obj = actualObject
    } else {
         if meta, err := meta.Accessor(obj); err == nil && o.EnforceNamespace {
            meta.SetNamespace(o.Namespace)
        }
    }

    return o.PrintObj(obj, o.Out)
}
{{< /highlight >}}
