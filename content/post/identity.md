---
title: "identity"
date: 2020-02-10T08:30:30+08:00
draft: true
categories: ["技术"]
tags: ["docker"]
---
窗间梅熟落蒂，墙下笋成出林。连雨不知春去，一晴方觉夏深。——范成大《喜晴》
<!--more-->
# 变量常量
{{< highlight go "linenos=inline" >}}
// github.com/moby/buildkit/identity/randomid.go
package identity

import (
	cryptorand "crypto/rand"
	"fmt"
	"io"
	"math/big"
)

var (
	// idReader is used for random id generation. This declaration allows us to
	// replace it for testing.
	idReader = cryptorand.Reader
)

// parameters for random identifier generation. We can tweak this when there is
// time for further analysis.
const (
	randomIDEntropyBytes = 17
	randomIDBase         = 36

	// To ensure that all identifiers are fixed length, we make sure they
	// get padded out or truncated to 25 characters.
	//
	// For academics,  f5lxx1zz5pnorynqglhzmsp33  == 2^128 - 1. This value
	// was calculated from floor(log(2^128-1, 36)) + 1.
	//
	// While 128 bits is the largest whole-byte size that fits into 25
	// base-36 characters, we generate an extra byte of entropy to fill
	// in the high bits, which would otherwise be 0. This gives us a more
	// even distribution of the first character.
	//
	// See http://mathworld.wolfram.com/NumberLength.html for more information.
	maxRandomIDLength = 25
)

{{< /highlight >}}
# NewID
{{< highlight go "linenos=inline" >}}
// NewID generates a new identifier for use where random identifiers with low
// collision probability are required.
//
// With the parameters in this package, the generated identifier will provide
// ~129 bits of entropy encoded with base36. Leading padding is added if the
// string is less 25 bytes. We do not intend to maintain this interface, so
// identifiers should be treated opaquely.
func NewID() string {
	var p [randomIDEntropyBytes]byte

	if _, err := io.ReadFull(idReader, p[:]); err != nil {
		panic(fmt.Errorf("failed to read random bytes: %v", err))
	}

	p[0] |= 0x80 // set high bit to avoid the need for padding
	return (&big.Int{}).SetBytes(p[:]).Text(randomIDBase)[1 : maxRandomIDLength+1]
}
{{< /highlight >}}

# rand.Reader
{{< highlight go "linenos=inline" >}}
import "crypto/rand"

var Reader io.Reader
Reader is a global, shared instance of a cryptographically(adv. 密码地；用暗号地)
secure random number generator.

On Linux and FreeBSD, Reader uses getrandom(2) if available, /dev/urandom
otherwise. On OpenBSD, Reader uses getentropy(2). On other
Unix-like systems, Reader reads from /dev/urandom. On Windows systems, Reader
uses the CryptGenRandom API. On Wasm, Reader uses the Web Crypto API.
{{< /highlight >}}

# io.ReadFull
{{< highlight go "linenos=inline" >}}
import "io"
func ReadFull(r Reader, buf []byte) (n int, err error)  

ReadFull reads exactly len(buf) bytes from r into buf. It returns the number of bytes copied and an error if fewer bytes were read. The error is EOF only if no bytes were read. If an EOF happens after reading some but not all the bytes, ReadFull returns ErrUnexpectedEOF. On return, n == len(buf) if and only if err == nil. If r returns an error having read at least len(buf) bytes, the error is dropped.
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
{{< /highlight >}}
