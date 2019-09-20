---
title: "k8s.io-cli-runtime-pkg-genericclioptions-record_flags"
date: 2019-09-07T17:16:42+08:00
draft: true
categories: [""]
tags: [""]
---
# 简介
<!--more-->
# 文件路径
{{< highlight go "linenos=inline" >}}
k8s.io/cli-runtime/pkg/genericclioptions/record_flags.go
{{< /highlight >}}

package genericclioptions

import (
	"os"
	"path/filepath"
	"strings"

	"github.com/evanphx/json-patch"
	"github.com/spf13/cobra"
	"github.com/spf13/pflag"

	"k8s.io/apimachinery/pkg/api/meta"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/util/json"
)

// ChangeCauseAnnotation is the annotation indicating a guess at "why" something was changed
const ChangeCauseAnnotation = "kubernetes.io/change-cause"

# RecordFlags
RecordFlags用于和--record选项有关的命令行操作
{{< highlight go "linenos=inline" >}}
// RecordFlags contains all flags associated with the "--record" operation
type RecordFlags struct {
	// Record indicates the state of the recording flag.  It is a pointer so a caller can opt out or rebind
	Record *bool

	changeCause string
}
{{< /highlight >}}

// ToRecorder returns a ChangeCause recorder if --record=false was not
// explicitly given by the user
func (f *RecordFlags) ToRecorder() (Recorder, error) {
	if f == nil {
		return NoopRecorder{}, nil
	}

	shouldRecord := false
	if f.Record != nil {
		shouldRecord = *f.Record
	}

	// if flag was explicitly set to false by the user,
	// do not record
	if !shouldRecord {
		return NoopRecorder{}, nil
	}

	return &ChangeCauseRecorder{
		changeCause: f.changeCause,
	}, nil
}

// Complete is called before the command is run, but after it is invoked to finish the state of the struct before use.
func (f *RecordFlags) Complete(cmd *cobra.Command) error {
	if f == nil {
		return nil
	}

	f.changeCause = parseCommandArguments(cmd)
	return nil
}

func (f *RecordFlags) CompleteWithChangeCause(cause string) error {
	if f == nil {
		return nil
	}

	f.changeCause = cause
	return nil
}

// AddFlags binds the requested flags to the provided flagset
// TODO have this only take a flagset
func (f *RecordFlags) AddFlags(cmd *cobra.Command) {
	if f == nil {
		return
	}

	if f.Record != nil {
		cmd.Flags().BoolVar(f.Record, "record", *f.Record, "Record current kubectl command in the resource annotation. If set to false, do not record the command. If set to true, record the command. If not set, default to updating the existing annotation value only if one already exists.")
	}
}

# NewRecordFlags
创建命令行操作记录器选项
{{< highlight go "linenos=inline" >}}
// NewRecordFlags provides a RecordFlags with reasonable default values set for use
func NewRecordFlags() *RecordFlags {
	record := false

	return &RecordFlags{
		Record: &record,
	}
}
{{< /highlight >}}

# Recorder
Recorder定义了一个接口，该接口的实例可以记录对象为什么变化
{{< highlight go "linenos=inline" >}}
// Recorder is used to record why a runtime.Object was changed in an annotation.
type Recorder interface {
	// Record records why a runtime.Object was changed in an annotation.
	Record(runtime.Object) error
	MakeRecordMergePatch(runtime.Object) ([]byte, error)
}
{{< /highlight >}}

# NoopRecorder
空记录器
{{< highlight go "linenos=inline" >}}
// NoopRecorder does nothing.  It is a "do nothing" that can be returned so code doesn't switch on it.
type NoopRecorder struct{}
{{< /highlight >}}

// Record implements Recorder
func (r NoopRecorder) Record(obj runtime.Object) error {
	return nil
}

// MakeRecordMergePatch implements Recorder
func (r NoopRecorder) MakeRecordMergePatch(obj runtime.Object) ([]byte, error) {
	return nil, nil
}

// ChangeCauseRecorder annotates a "change-cause" to an input runtime object
type ChangeCauseRecorder struct {
	changeCause string
}

// Record annotates a "change-cause" to a given info if either "shouldRecord" is true,
// or the resource info previously contained a "change-cause" annotation.
func (r *ChangeCauseRecorder) Record(obj runtime.Object) error {
	accessor, err := meta.Accessor(obj)
	if err != nil {
		return err
	}
	annotations := accessor.GetAnnotations()
	if annotations == nil {
		annotations = make(map[string]string)
	}
	annotations[ChangeCauseAnnotation] = r.changeCause
	accessor.SetAnnotations(annotations)
	return nil
}

// MakeRecordMergePatch produces a merge patch for updating the recording annotation.
func (r *ChangeCauseRecorder) MakeRecordMergePatch(obj runtime.Object) ([]byte, error) {
	// copy so we don't mess with the original
	objCopy := obj.DeepCopyObject()
	if err := r.Record(objCopy); err != nil {
		return nil, err
	}

	oldData, err := json.Marshal(obj)
	if err != nil {
		return nil, err
	}
	newData, err := json.Marshal(objCopy)
	if err != nil {
		return nil, err
	}

	return jsonpatch.CreateMergePatch(oldData, newData)
}

// parseCommandArguments will stringify and return all environment arguments ie. a command run by a client
// using the factory.
// Set showSecrets false to filter out stuff like secrets.
func parseCommandArguments(cmd *cobra.Command) string {
	if len(os.Args) == 0 {
		return ""
	}

	flags := ""
	parseFunc := func(flag *pflag.Flag, value string) error {
		flags = flags + " --" + flag.Name
		if set, ok := flag.Annotations["classified"]; !ok || len(set) == 0 {
			flags = flags + "=" + value
		} else {
			flags = flags + "=CLASSIFIED"
		}
		return nil
	}
	var err error
	err = cmd.Flags().ParseAll(os.Args[1:], parseFunc)
	if err != nil || !cmd.Flags().Parsed() {
		return ""
	}

	args := ""
	if arguments := cmd.Flags().Args(); len(arguments) > 0 {
		args = " " + strings.Join(arguments, " ")
	}

	base := filepath.Base(os.Args[0])
	return base + args + flags
}

