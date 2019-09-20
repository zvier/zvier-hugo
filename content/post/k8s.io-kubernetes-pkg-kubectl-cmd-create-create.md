---
title: "k8s kubectl create"
date: 2019-09-07T10:41:00+08:00
draft: true
categories: [""]
tags: ["k8s"]
---
# 简述
create.go中实现的是kubectl create操作的公共部分
<!--more-->

# 文件路径
{{< highlight go "linenos=inline" >}}
k8s.io/kubernetes/pkg/kubectl/cmd/create/create.go
{{< /highlight >}}

# CreateOptions
CreateOptions是<code>kubectl create</code>子命令行用于存储相关选项的数据结构  
{{< highlight go "linenos=inline" >}}
import (
    "k8s.io/cli-runtime/pkg/genericclioptions"
    "k8s.io/cli-runtime/pkg/resource"
    kruntime "k8s.io/apimachinery/pkg/runtime"
)
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
// CreateOptions is the commandline options for 'create' sub command
type CreateOptions struct {
	PrintFlags  *genericclioptions.PrintFlags
	RecordFlags *genericclioptions.RecordFlags

	DryRun bool

	FilenameOptions  resource.FilenameOptions
	Selector         string
	EditBeforeCreate bool
	Raw              string

	Recorder genericclioptions.Recorder
	PrintObj func(obj kruntime.Object) error

	genericclioptions.IOStreams
}
{{< /highlight >}}
## 引用说明
1. 命令行输出格式控制: [genericclioptions.PrintFlags](http://www.zvier.top/post/k8s.io-cli-runtime-pkg-genericclioptions-print_flags/#printflags)  
2. 命令行操作记录控制: [genericclioptions.RecordFlags](http://www.zvier.top/post/k8s.io-cli-runtime-pkg-genericclioptions-record_flags/#recordflags)  
4. 对象变化记录器: [genericclioptions.Recorder](http://www.zvier.top/post/k8s.io-cli-runtime-pkg-genericclioptions-record_flags/#recorder)
6. 标准输入输出和错误流: [genericclioptions.IOStreams](http://www.zvier.top/post/k8s.io-cli-runtime-pkg-genericclioptions-io_options/#iostreams)
3. 资源文件选项: [resource.FilenameOptions](http://www.zvier.top/post/k8s.io-cli-runtime-pkg-resource-builder/#filenameoptions)
4. API资源对象基类: [kruntime.Object](http://www.zvier.top/post/k8s.io-apimachinery-pkg-runtime-interfaces/#object)

# 变量声明
基于templates包声明来kubectl create的使用和示例说明的两个变量，分别为createLong和createExample  
{{< highlight go "linenos=inline" >}}
import(
    "k8s.io/kubernetes/pkg/kubectl/util/templates"
    "k8s.io/kubernetes/pkg/kubectl/util/i18n"
)
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
var (
	createLong = templates.LongDesc(i18n.T(`
		Create a resource from a file or from stdin.

		JSON and YAML formats are accepted.`))

	createExample = templates.Examples(i18n.T(`
		# Create a pod using the data in pod.json.
		kubectl create -f ./pod.json

		# Create a pod based on the JSON passed into stdin.
		cat pod.json | kubectl create -f -

		# Edit the data in docker-registry.yaml in JSON then create the resource using the edited data.
		kubectl create -f docker-registry.yaml --edit -o json`))
)
{{< /highlight >}}

## 引用说明
1. 描述信息格式化: [templates.LongDesc](http://www.zvier.top/post/k8s.io-kubernetes-pkg-kubectl-util-templates-normalizers/#longdesc)
2. 示例说明格式化: [templates.Examples](http://www.zvier.top/post/k8s.io-kubernetes-pkg-kubectl-util-templates-normalizers/#examples)
3. 字符串转换: [i18n.T](http://www.zvier.top/post/k8s.io-kubernetes-pkg-kubectl-util-i18n-i18n/#t)

# NewCreateOptions
NewCreateOptions用于生成和初始化一个CreateOptions对象
{{< highlight go "linenos=inline" >}}
// NewCreateOptions returns an initialized CreateOptions instance
func NewCreateOptions(ioStreams genericclioptions.IOStreams) *CreateOptions {
	return &CreateOptions{
		PrintFlags:  genericclioptions.NewPrintFlags("created").WithTypeSetter(scheme.Scheme),
		RecordFlags: genericclioptions.NewRecordFlags(),

		Recorder: genericclioptions.NoopRecorder{},

		IOStreams: ioStreams,
	}
}
{{< /highlight >}}
## 引用说明
1. 创建打印选项: [genericclioptions.NewPrintFlags](http://www.zvier.top/post/k8s.io-cli-runtime-pkg-genericclioptions-print_flags/#newprintflags)
2. 创建命令行操作记录器选项: [genericclioptions.NewRecordFlags](http://www.zvier.top/post/k8s.io-cli-runtime-pkg-genericclioptions-record_flags/#newrecordflags)
3. 空记录器: [genericclioptions.NoopRecorder](http://www.zvier.top/post/k8s.io-cli-runtime-pkg-genericclioptions-record_flags/#nooprecorder)

# NewCmdCreate
NewCmdCreate创建create子命令行
{{< highlight go "linenos=inline" >}}
import (
    "k8s.io/kubernetes/pkg/kubectl/util/i18n"
    cmdutil "k8s.io/kubernetes/pkg/kubectl/cmd/util"
)
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
// NewCmdCreate returns new initialized instance of create sub command
func NewCmdCreate(f cmdutil.Factory, ioStreams genericclioptions.IOStreams) *cobra.Command {
    o := NewCreateOptions(ioStreams)

    cmd := &cobra.Command{
        Use:                   "create -f FILENAME",
        DisableFlagsInUseLine: true,
        Short:                 i18n.T("Create a resource from a file or from stdin."),
        Long:                  createLong,
        Example:               createExample,
        Run: func(cmd *cobra.Command, args []string) {
            if cmdutil.IsFilenameSliceEmpty(o.FilenameOptions.Filenames, o.FilenameOptions.Kustomize) {
                ioStreams.ErrOut.Write([]byte("Error: must specify one of -f and -k\n\n"))
                defaultRunFunc := cmdutil.DefaultSubCommandRun(ioStreams.ErrOut)
                defaultRunFunc(cmd, args)
                return
            }
            cmdutil.CheckErr(o.Complete(f, cmd))
            cmdutil.CheckErr(o.ValidateArgs(cmd, args))
            cmdutil.CheckErr(o.RunCreate(f, cmd))
        },
    }
    // bind flag structs
    o.RecordFlags.AddFlags(cmd)

    usage := "to use to create the resource"
    cmdutil.AddFilenameOptionFlags(cmd, &o.FilenameOptions, usage)
    cmdutil.AddValidateFlags(cmd)
    cmd.Flags().BoolVar(&o.EditBeforeCreate, "edit", o.EditBeforeCreate, "Edit the API resource before creating")
    cmd.Flags().Bool("windows-line-endings", runtime.GOOS == "windows",
        "Only relevant if --edit=true. Defaults to the line ending native to your platform.")
    cmdutil.AddApplyAnnotationFlags(cmd)
    cmdutil.AddDryRunFlag(cmd)
    cmd.Flags().StringVarP(&o.Selector, "selector", "l", o.Selector, "Selector (label query) to filter on, supports '=', '==', and '!='.(e.g. -l key1=value1,key2=value2)    ")
     cmd.Flags().StringVar(&o.Raw, "raw", o.Raw, "Raw URI to POST to the server.  Uses the transport specified by the kubeconfig file.")

     o.PrintFlags.AddFlags(cmd)

     // create subcommands
     cmd.AddCommand(NewCmdCreateNamespace(f, ioStreams))
     cmd.AddCommand(NewCmdCreateQuota(f, ioStreams))
     cmd.AddCommand(NewCmdCreateSecret(f, ioStreams))
     cmd.AddCommand(NewCmdCreateConfigMap(f, ioStreams))
     cmd.AddCommand(NewCmdCreateServiceAccount(f, ioStreams))
     cmd.AddCommand(NewCmdCreateService(f, ioStreams))
     cmd.AddCommand(NewCmdCreateDeployment(f, ioStreams))
     cmd.AddCommand(NewCmdCreateClusterRole(f, ioStreams))
     cmd.AddCommand(NewCmdCreateClusterRoleBinding(f, ioStreams))
     cmd.AddCommand(NewCmdCreateRole(f, ioStreams))
     cmd.AddCommand(NewCmdCreateRoleBinding(f, ioStreams))
     cmd.AddCommand(NewCmdCreatePodDisruptionBudget(f, ioStreams))
     cmd.AddCommand(NewCmdCreatePriorityClass(f, ioStreams))
     cmd.AddCommand(NewCmdCreateJob(f, ioStreams))
     cmd.AddCommand(NewCmdCreateCronJob(f, ioStreams))
     return cmd
 }
{{< /highlight >}}
## 引用说明
1.  [cmdutil.Factory](http://www.zvier.top/post/k8s.io-kubernetes-pkg-kubectl-cmd-util-factory/#factory)
2. cmdutil.IsFilenameSliceEmpty
3. cmdutil.DefaultSubCommandRun
4. cmdutil.CheckErr
5. o.Complete
6. o.ValidateArgs
7. o.RunCreate

# CreateOptions.ValidateArgs
ValidateArgs主要用于校验create选项的有效性
{{< highlight go "linenos=inline" >}}
// ValidateArgs makes sure there is no discrepency(矛盾，不一致) in command options
func (o *CreateOptions) ValidateArgs(cmd *cobra.Command, args []string) error {
	if len(args) != 0 {
		return cmdutil.UsageErrorf(cmd, "Unexpected args: %v", args)
	}
	if len(o.Raw) > 0 {
		if o.EditBeforeCreate {
			return cmdutil.UsageErrorf(cmd, "--raw and --edit are mutually exclusive")
		}
		if len(o.FilenameOptions.Filenames) != 1 {
			return cmdutil.UsageErrorf(cmd, "--raw can only use a single local file or stdin")
		}
		if strings.Index(o.FilenameOptions.Filenames[0], "http://") == 0 || strings.Index(o.FilenameOptions.Filenames[0], "https://") == 0 {
			return cmdutil.UsageErrorf(cmd, "--raw cannot read from a url")
		}
		if o.FilenameOptions.Recursive {
			return cmdutil.UsageErrorf(cmd, "--raw and --recursive are mutually exclusive")
		}
		if len(o.Selector) > 0 {
			return cmdutil.UsageErrorf(cmd, "--raw and --selector (-l) are mutually exclusive")
		}
		if len(cmdutil.GetFlagString(cmd, "output")) > 0 {
			return cmdutil.UsageErrorf(cmd, "--raw and --output are mutually exclusive")
		}
		if _, err := url.ParseRequestURI(o.Raw); err != nil {
			return cmdutil.UsageErrorf(cmd, "--raw must be a valid URL path: %v", err)
		}
	}

	return nil
}
{{< /highlight >}}

# CreateOptions.Complete
CreateOptions.Complete用户完善CreateOptions的选项
{{< highlight go "linenos=inline" >}}
// Complete completes all the required options
func (o *CreateOptions) Complete(f cmdutil.Factory, cmd *cobra.Command) error {
	var err error
	o.RecordFlags.Complete(cmd)
	o.Recorder, err = o.RecordFlags.ToRecorder()
	if err != nil {
		return err
	}

	o.DryRun = cmdutil.GetDryRunFlag(cmd)

	if o.DryRun {
		o.PrintFlags.Complete("%s (dry run)")
	}
	printer, err := o.PrintFlags.ToPrinter()
	if err != nil {
		return err
	}

	o.PrintObj = func(obj kruntime.Object) error {
		return printer.PrintObj(obj, o.Out)
	}

	return nil
}
{{< /highlight >}}

# CreateOptions.RunCreate
CreateOptions.RunCreate根据选项执行，在RunCreate中，所有重要的参数都是由Factory生产
{{< highlight go "linenos=inline" >}}
// RunCreate performs the creation
func (o *CreateOptions) RunCreate(f cmdutil.Factory, cmd *cobra.Command) error {
	// raw only makes sense for a single file resource multiple objects aren't likely to do what you want.
	// the validator enforces this, so
	if len(o.Raw) > 0 {
		return o.raw(f)
	}

	if o.EditBeforeCreate {
		return RunEditOnCreate(f, o.PrintFlags, o.RecordFlags, o.IOStreams, cmd, &o.FilenameOptions)
	}
	schema, err := f.Validator(cmdutil.GetFlagBool(cmd, "validate"))
	if err != nil {
		return err
	}

	cmdNamespace, enforceNamespace, err := f.ToRawKubeConfigLoader().Namespace()
	if err != nil {
		return err
	}

	r := f.NewBuilder().
		Unstructured().
		Schema(schema).
		ContinueOnError().
		NamespaceParam(cmdNamespace).DefaultNamespace().
		FilenameParam(enforceNamespace, &o.FilenameOptions).
		LabelSelectorParam(o.Selector).
		Flatten().
		Do()
	err = r.Err()
	if err != nil {
		return err
	}

	count := 0
	err = r.Visit(func(info *resource.Info, err error) error {
		if err != nil {
			return err
		}
		if err := kubectl.CreateOrUpdateAnnotation(cmdutil.GetFlagBool(cmd, cmdutil.ApplyAnnotationsFlag), info.Object, scheme.DefaultJSONEncoder()); err != nil {
			return cmdutil.AddSourceToErr("creating", info.Source, err)
		}

		if err := o.Recorder.Record(info.Object); err != nil {
			klog.V(4).Infof("error recording current command: %v", err)
		}

		if !o.DryRun {
			if err := createAndRefresh(info); err != nil {
				return cmdutil.AddSourceToErr("creating", info.Source, err)
			}
		}

		count++

		return o.PrintObj(info.Object)
	})
	if err != nil {
		return err
	}
	if count == 0 {
		return fmt.Errorf("no objects passed to create")
	}
	return nil
}
{{< /highlight >}}

// raw makes a simple HTTP request to the provided path on the server using the default
// credentials.
func (o *CreateOptions) raw(f cmdutil.Factory) error {
	restClient, err := f.RESTClient()
	if err != nil {
		return err
	}

	var data io.ReadCloser
	if o.FilenameOptions.Filenames[0] == "-" {
		data = os.Stdin
	} else {
		data, err = os.Open(o.FilenameOptions.Filenames[0])
		if err != nil {
			return err
		}
	}
	// TODO post content with stream.  Right now it ignores body content
	result := restClient.Post().RequestURI(o.Raw).Body(data).Do()
	if err := result.Error(); err != nil {
		return err
	}
	body, err := result.Raw()
	if err != nil {
		return err
	}

	fmt.Fprintf(o.Out, "%v", string(body))
	return nil
}

// RunEditOnCreate performs edit on creation
func RunEditOnCreate(f cmdutil.Factory, printFlags *genericclioptions.PrintFlags, recordFlags *genericclioptions.RecordFlags, ioStreams genericclioptions.IOStreams, cmd *cobra.Command, options *resource.FilenameOptions) error {
	editOptions := editor.NewEditOptions(editor.EditBeforeCreateMode, ioStreams)
	editOptions.FilenameOptions = *options
	editOptions.ValidateOptions = cmdutil.ValidateOptions{
		EnableValidation: cmdutil.GetFlagBool(cmd, "validate"),
	}
	editOptions.PrintFlags = printFlags
	editOptions.ApplyAnnotation = cmdutil.GetFlagBool(cmd, cmdutil.ApplyAnnotationsFlag)
	editOptions.RecordFlags = recordFlags

	err := editOptions.Complete(f, []string{}, cmd)
	if err != nil {
		return err
	}
	return editOptions.Run()
}

// createAndRefresh creates an object from input info and refreshes info with that object
func createAndRefresh(info *resource.Info) error {
	obj, err := resource.NewHelper(info.Client, info.Mapping).Create(info.Namespace, true, info.Object, nil)
	if err != nil {
		return err
	}
	info.Refresh(obj, true)
	return nil
}

// NameFromCommandArgs is a utility function for commands that assume the first argument is a resource name
func NameFromCommandArgs(cmd *cobra.Command, args []string) (string, error) {
	argsLen := cmd.ArgsLenAtDash()
	// ArgsLenAtDash returns -1 when -- was not specified
	if argsLen == -1 {
		argsLen = len(args)
	}
	if argsLen != 1 {
		return "", cmdutil.UsageErrorf(cmd, "exactly one NAME is required, got %d", argsLen)
	}
	return args[0], nil
}

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

// NewCreateSubcommandOptions returns initialized CreateSubcommandOptions
func NewCreateSubcommandOptions(ioStreams genericclioptions.IOStreams) *CreateSubcommandOptions {
	return &CreateSubcommandOptions{
		PrintFlags: genericclioptions.NewPrintFlags("created").WithTypeSetter(scheme.Scheme),
		IOStreams:  ioStreams,
	}
}

// Complete completes all the required options
func (o *CreateSubcommandOptions) Complete(f cmdutil.Factory, cmd *cobra.Command, args []string, generator generate.StructuredGenerator) error {
	name, err := NameFromCommandArgs(cmd, args)
	if err != nil {
		return err
	}

	o.Name = name
	o.StructuredGenerator = generator
	o.DryRun = cmdutil.GetDryRunFlag(cmd)
	o.CreateAnnotation = cmdutil.GetFlagBool(cmd, cmdutil.ApplyAnnotationsFlag)

	if o.DryRun {
		o.PrintFlags.Complete("%s (dry run)")
	}
	printer, err := o.PrintFlags.ToPrinter()
	if err != nil {
		return err
	}

	o.PrintObj = func(obj kruntime.Object, out io.Writer) error {
		return printer.PrintObj(obj, out)
	}

	o.Namespace, o.EnforceNamespace, err = f.ToRawKubeConfigLoader().Namespace()
	if err != nil {
		return err
	}

	o.DynamicClient, err = f.DynamicClient()
	if err != nil {
		return err
	}

	o.Mapper, err = f.ToRESTMapper()
	if err != nil {
		return err
	}

	return nil
}

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
