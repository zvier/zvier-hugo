---
title: "k8s.io-cli-runtime-pkg-genericclioptions-print_flags"
date: 2019-09-07T17:07:30+08:00
draft: true
categories: [""]
tags: ["k8s"]
---
# 简述
<!--more-->

# 文件路径
{{< highlight go "linenos=inline" >}}
k8s.io/cli-runtime/pkg/genericclioptions/print_flags.go
{{< /highlight >}}
import (
	"fmt"
	"sort"
	"strings"

	"github.com/spf13/cobra"

	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/cli-runtime/pkg/printers"
)

type NoCompatiblePrinterError struct {
	OutputFormat   *string
	AllowedFormats []string
	Options        interface{}
}

func (e NoCompatiblePrinterError) Error() string {
	output := ""
	if e.OutputFormat != nil {
		output = *e.OutputFormat
	}

	sort.Strings(e.AllowedFormats)
	return fmt.Sprintf("unable to match a printer suitable for the output format %q, allowed formats are: %s", output, strings.Join(e.AllowedFormats, ","))
}

func IsNoCompatiblePrinterError(err error) bool {
	if err == nil {
		return false
	}

	_, ok := err.(NoCompatiblePrinterError)
	return ok
}

# PrintFlags
PrintFlags用于控制命令行的输出形式，json还是yaml
{{< highlight go "linenos=inline" >}}
// PrintFlags composes common printer flag structs
// used across all commands, and provides a method
// of retrieving a known printer based on flag values provided.
type PrintFlags struct {
	JSONYamlPrintFlags   *JSONYamlPrintFlags
	NamePrintFlags       *NamePrintFlags
	TemplatePrinterFlags *KubeTemplatePrintFlags

	TypeSetterPrinter *printers.TypeSetterPrinter

	OutputFormat *string

	// OutputFlagSpecified indicates whether the user specifically requested a certain kind of output.
	// Using this function allows a sophisticated caller to change the flag binding logic if they so desire.
	OutputFlagSpecified func() bool
}
{{< /highlight >}}

func (f *PrintFlags) Complete(successTemplate string) error {
	return f.NamePrintFlags.Complete(successTemplate)
}

func (f *PrintFlags) AllowedFormats() []string {
	ret := []string{}
	ret = append(ret, f.JSONYamlPrintFlags.AllowedFormats()...)
	ret = append(ret, f.NamePrintFlags.AllowedFormats()...)
	ret = append(ret, f.TemplatePrinterFlags.AllowedFormats()...)
	return ret
}

func (f *PrintFlags) ToPrinter() (printers.ResourcePrinter, error) {
	outputFormat := ""
	if f.OutputFormat != nil {
		outputFormat = *f.OutputFormat
	}
	// For backwards compatibility we want to support a --template argument given, even when no --output format is provided.
	// If no explicit output format has been provided via the --output flag, fallback
	// to honoring the --template argument.
	templateFlagSpecified := f.TemplatePrinterFlags != nil &&
		f.TemplatePrinterFlags.TemplateArgument != nil &&
		len(*f.TemplatePrinterFlags.TemplateArgument) > 0
	outputFlagSpecified := f.OutputFlagSpecified != nil && f.OutputFlagSpecified()
	if templateFlagSpecified && !outputFlagSpecified {
		outputFormat = "go-template"
	}

	if f.JSONYamlPrintFlags != nil {
		if p, err := f.JSONYamlPrintFlags.ToPrinter(outputFormat); !IsNoCompatiblePrinterError(err) {
			return f.TypeSetterPrinter.WrapToPrinter(p, err)
		}
	}

	if f.NamePrintFlags != nil {
		if p, err := f.NamePrintFlags.ToPrinter(outputFormat); !IsNoCompatiblePrinterError(err) {
			return f.TypeSetterPrinter.WrapToPrinter(p, err)
		}
	}

	if f.TemplatePrinterFlags != nil {
		if p, err := f.TemplatePrinterFlags.ToPrinter(outputFormat); !IsNoCompatiblePrinterError(err) {
			return f.TypeSetterPrinter.WrapToPrinter(p, err)
		}
	}

	return nil, NoCompatiblePrinterError{OutputFormat: f.OutputFormat, AllowedFormats: f.AllowedFormats()}
}

func (f *PrintFlags) AddFlags(cmd *cobra.Command) {
	f.JSONYamlPrintFlags.AddFlags(cmd)
	f.NamePrintFlags.AddFlags(cmd)
	f.TemplatePrinterFlags.AddFlags(cmd)

	if f.OutputFormat != nil {
		cmd.Flags().StringVarP(f.OutputFormat, "output", "o", *f.OutputFormat, fmt.Sprintf("Output format. One of: %s.", strings.Join(f.AllowedFormats(), "|")))
		if f.OutputFlagSpecified == nil {
			f.OutputFlagSpecified = func() bool {
				return cmd.Flag("output").Changed
			}
		}
	}
}

// WithDefaultOutput sets a default output format if one is not provided through a flag value
func (f *PrintFlags) WithDefaultOutput(output string) *PrintFlags {
	f.OutputFormat = &output
	return f
}

// WithTypeSetter sets a wrapper than will surround the returned printer with a printer to type resources
func (f *PrintFlags) WithTypeSetter(scheme *runtime.Scheme) *PrintFlags {
	f.TypeSetterPrinter = printers.NewTypeSetter(scheme)
	return f
}

# NewPrintFlags
创建和初始化一个PrintFlags
{{< highlight go "linenos=inline" >}}
func NewPrintFlags(operation string) *PrintFlags {
	outputFormat := ""

	return &PrintFlags{
		OutputFormat: &outputFormat,

		JSONYamlPrintFlags:   NewJSONYamlPrintFlags(),
		NamePrintFlags:       NewNamePrintFlags(operation),
		TemplatePrinterFlags: NewKubeTemplatePrintFlags(),
	}
}
{{< /highlight >}}
