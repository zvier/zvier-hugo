---
title: "Golant Selectcase"
date: 2019-06-24T20:57:56+08:00
draft: true
categories: [""]
tags: ["golang"]
---

{{< highlight go "linenos=inline" >}}
// A SelectCase describes a single case in a select operation.
// The kind of case depends on Dir, the communication direction.
//
// If Dir is SelectDefault, the case represents a default case.
// Chan and Send must be zero Values.
//
// If Dir is SelectSend, the case represents a send operation.
// Normally Chan's underlying value must be a channel, and Send's underlying value must be
// assignable to the channel's element type. As a special case, if Chan is a zero Value,
// then the case is ignored, and the field Send will also be ignored and may be either zero
// or non-zero.
//
// If Dir is SelectRecv, the case represents a receive operation.
// Normally Chan's underlying value must be a channel and Send must be a zero Value.
// If Chan is a zero Value, then the case is ignored, but Send must still be a zero Value.
// When a receive operation is selected, the received Value is returned by Select.
//
type SelectCase struct {
    Dir  SelectDir // direction of case
    Chan Value     // channel to use (for send or receive)
    Send Value     // value to send (for send)
}

{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
// Select executes a select operation described by the list of cases.
// Like the Go select statement, it blocks until at least one of the cases
// can proceed, makes a uniform pseudo-random choice,
// and then executes that case. It returns the index of the chosen case
// and, if that case was a receive operation, the value received and a
// boolean indicating whether the value corresponds to a send on the channel
// (as opposed to a zero value received because the channel is closed).

func Select(cases []SelectCase) (chosen int, recv Value, recvOK bool) {
    // NOTE: Do not trust that caller is not modifying cases data underfoot.
    // The range is safe because the caller cannot modify our copy of the len
    // and each iteration makes its own copy of the value c.
    runcases := make([]runtimeSelect, len(cases))
    haveDefault := false
    for i, c := range cases {
        rc := &runcases[i]
        rc.dir = c.Dir
        switch c.Dir {
        default:
            panic("reflect.Select: invalid Dir")

        case SelectDefault: // default
            if haveDefault {
                panic("reflect.Select: multiple default cases")
            }
            haveDefault = true
            if c.Chan.IsValid() {
                panic("reflect.Select: default case has Chan value")
            }
            if c.Send.IsValid() {
                panic("reflect.Select: default case has Send value")
            }
        case SelectSend:
            ch := c.Chan
            if !ch.IsValid() {
                break
            }
            ch.mustBe(Chan)
            ch.mustBeExported()
            tt := (*chanType)(unsafe.Pointer(ch.typ))
            if ChanDir(tt.dir)&SendDir == 0 {
                panic("reflect.Select: SendDir case using recv-only channel")
            }
            rc.ch = ch.pointer()
            rc.typ = &tt.rtype
            v := c.Send
            if !v.IsValid() {
                panic("reflect.Select: SendDir case missing Send value")
            }
            v.mustBeExported()
            v = v.assignTo("reflect.Select", tt.elem, nil)
            if v.flag&flagIndir != 0 {
                rc.val = v.ptr
            } else {
                rc.val = unsafe.Pointer(&v.ptr)
            }
        case SelectRecv:
            if c.Send.IsValid() {
                panic("reflect.Select: RecvDir case has Send value")
            }
            ch := c.Chan
            if !ch.IsValid() {
                break
            }
            ch.mustBe(Chan)
            ch.mustBeExported()
            tt := (*chanType)(unsafe.Pointer(ch.typ))
            if ChanDir(tt.dir)&RecvDir == 0 {
                panic("reflect.Select: RecvDir case using send-only channel")
            }
            rc.ch = ch.pointer()
            rc.typ = &tt.rtype
            rc.val = unsafe_New(tt.elem)
        }
    }
    chosen, recvOK = rselect(runcases)
    if runcases[chosen].dir == SelectRecv {
        tt := (*chanType)(unsafe.Pointer(runcases[chosen].typ))
        t := tt.elem
        p := runcases[chosen].val
        fl := flag(t.Kind())
        if ifaceIndir(t) {
            recv = Value{t, p, fl | flagIndir}
        } else {
            recv = Value{t, *(*unsafe.Pointer)(p), fl}
        }
    }
    return chosen, recv, recvOK
}
{{< /highlight >}}
{{< highlight go "linenos=inline" >}}
   2 // rselect runs a select. It returns the index of the chosen case.
   3 // If the case was a receive, val is filled in with the received value.
   4 // The conventional OK bool indicates whether the receive corresponds
   5 // to a sent value.
   6 //go:noescape
   7 func rselect([]runtimeSelect) (chosen int, recvOK bool)

{{< /highlight >}}
