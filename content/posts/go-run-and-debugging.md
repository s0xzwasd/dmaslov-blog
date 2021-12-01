---
title: "How to fix decoding dwarf section info at offset 0x0: too short?"
date: 2021-12-01T16:04:32+03:00
draft: false
categories: Go
tags: [debugging, delve, go commands]
---

Difference between `go build` & `go run` and why the latter is not suitable for debugging. 

## Introduction

Let's assume you want to debug an application using Delve or other debugging tools (e.g. GoLand or VS Code built-in debuggers). As an example, I consider a simple HTTP-server with two handlers.

```go
package main

import (
	"fmt"
	"net/http"
)

func hello(w http.ResponseWriter, req *http.Request) {
	fmt.Fprintf(w, "hello\n")
}

func headers(w http.ResponseWriter, req *http.Request) {
	for name, headers := range req.Header {
		for _, h := range headers {
			fmt.Fprintf(w, "%v: %v\n", name, h)
		}
	}
}

func main() {
	http.HandleFunc("/hello", hello)
	http.HandleFunc("/headers", headers)

	http.ListenAndServe(":8080", nil)
}
```

A code snippet is pretty simple, it has `hello()` and `headers()` functions and listening for request on [http://localhost:8080/hello](http://localhost:8080/hello) and [http://localhost:8080/headers](http://localhost:8080/headers).

In some time, you might want to debug your application. It is a trivial part in Go world, isn't it?

- Run an application via `go run main.go` command.
- Install Delve using `go install github.com/go-delve/delve/cmd/dlv@latest` if Go version is higher than 1.16.
- Search for a PID via `ps`, then attach Delve debugger: `dlv attach pid [executable] [flags]`.

However, you can get `could not attach to pid <ID>: decoding dwarf section info at offset 0x0: too short` error. What happened?

## Difference between `go run` and `go build`

A bit of googling about difference between `go run` and `go build` produces the most voted answer on [StackOverflow](https://stackoverflow.com/a/28881839/13576700):

> go run is just a shortcut for compiling then running in a single step. While it is useful for development you should generally build it and run the binary directly when using it in production.

Sounds reasonable, but why I can't debug my application then?

## Secrets of `go run`

To understand the reasons, firstly, we have to know how `go run main.go` command works under the hood. It is where `-x` flag can help to get verbose output. Close the existing connection to HTTP-server, then run the application once using `go run -x main.go`. You should get the following:

```
WORK=/var/folders/z9/6jgs2k590rgg0r5wvh0hb44m0000gp/T/go-build2534330598
mkdir -p $WORK/b001/
cat >$WORK/b001/importcfg.link << 'EOF' # internal
...
<packagefile lines>
...
EOF
mkdir -p $WORK/b001/exe/
cd .
/usr/local/go/pkg/tool/darwin_amd64/link -o $WORK/b001/exe/main -importcfg $WORK/b001/importcfg.link -s -w -buildmode=exe -buildid=5D3UfnDtegsve1m8lzV0/ajtJPOzikoWHZgBSq-Cy/Hgg0qIzCay6Hig9bxCAr/5D3UfnDtegsve1m8lzV0 -extld=clang /Users/daniil.maslov/Library/Caches/go-build/2a/2aa5be369d2460bad7f61eca5da727de24978242d6085f9da7731baddb8e413b-d
$WORK/b001/exe/main
```

The process is simple, `go run` creates a new temporary directory, [link](https://pkg.go.dev/cmd/link) necessary files and combines them into an executable binary, then runs a binary. It is where the fun begins.

Consider link arguments under the microscope (full description of flags you can find [here](https://pkg.go.dev/cmd/link)):

- `-o`: write output to file, nothing special.
- `-importcfg`: internal part of building binary.
- `-buildmode`: another internal part.
- `-extld`: set the external linker, `clang` in our case.
- `-s`: omitting the symbol table and debug information. Sounds interesting.
- `-w` omitting the DWARF symbol table. Nothing special until you are aware about the DWARF. [Wikipedia](https://en.wikipedia.org/wiki/DWARF) says:

> DWARF is a widely used, standardized debugging data format. DWARF was originally designed along with Executable and Linkable Format (ELF), although it is independent of object file formats. The name is a medieval fantasy complement to "ELF" that had no official meaning, although the backronym "Debugging With Arbitrary Record Formats" has since been proposed

It seems we found a direction for further investigation. Let's take a look at `go build` documentation and execution to confirm our theory with `-s` and `-w`.

## Does `go build` have the same secrets?

TL;DL: it has. `go build` under the hood executes the same linker, but do it a bit differently. Execute `go build -x main.go` command:

```
WORK=/var/folders/z9/6jgs2k590rgg0r5wvh0hb44m0000gp/T/go-build4222066819
mkdir -p $WORK/b001/
cat >$WORK/b001/_gomod_.go << 'EOF' # internal
package main
import _ "unsafe"
//go:linkname __debug_modinfo__ runtime.modinfo
var __debug_modinfo__ = "0w\xaf\f\x92t\b\x02A\xe1\xc1\a\xe6\xd6\x18\xe6path\tcommand-line-arguments\nmod\tgo-1-17-project\t(devel)\t\n\xf92C1\x86\x18 r\x00\x82B\x10A\x16\xd8\xf2"
EOF
cat >$WORK/b001/importcfg << 'EOF' # internal
# import config
packagefile fmt=/usr/local/go/pkg/darwin_amd64/fmt.a
packagefile net/http=/Users/daniil.maslov/Library/Caches/go-build/5c/5cf3e56c325c109fe9da6dedfdbf3bdc6eab13be121e93028b85f4f6e6b437c4-d
packagefile runtime=/usr/local/go/pkg/darwin_amd64/runtime.a
EOF
cd /Users/daniil.maslov/Projects/Go/go-1-17-project
/usr/local/go/pkg/tool/darwin_amd64/compile -o $WORK/b001/_pkg_.a -trimpath "$WORK/b001=>" -p main -lang=go1.17 -complete -buildid KfpVyRzWQ0ASHdE8bXnx/KfpVyRzWQ0ASHdE8bXnx -goversion go1.17.2 -D _/Users/daniil.maslov/Projects/Go/go-1-17-project -importcfg $WORK/b001/importcfg -pack -c=4 ./main.go $WORK/b001/_gomod_.go
/usr/local/go/pkg/tool/darwin_amd64/buildid -w $WORK/b001/_pkg_.a # internal
cp $WORK/b001/_pkg_.a /Users/daniil.maslov/Library/Caches/go-build/8b/8bca05fdefbb1bfb48d924e8a2fc4bcb50c6b0873c4f5721b25355b3378b6396-d # internal
cat >$WORK/b001/importcfg.link << 'EOF' # internal
...
<packagefile lines>
...
EOF
mkdir -p $WORK/b001/exe/
cd .
/usr/local/go/pkg/tool/darwin_amd64/link -o $WORK/b001/exe/a.out -importcfg $WORK/b001/importcfg.link -buildmode=exe -buildid=D1x9DlretrpD_IbvR_-I/KfpVyRzWQ0ASHdE8bXnx/biWdPWPymJp1dLczFR0k/D1x9DlretrpD_IbvR_-I -extld=clang $WORK/b001/_pkg_.a
/usr/local/go/pkg/tool/darwin_amd64/buildid -w $WORK/b001/exe/a.out # internal
mv $WORK/b001/exe/a.out main
rm -r $WORK/b001/
```

All output is not interested to us, the latest lines are what we are looking for.

```
/usr/local/go/pkg/tool/darwin_amd64/link -o $WORK/b001/exe/a.out -importcfg $WORK/b001/importcfg.link -buildmode=exe -buildid=D1x9DlretrpD_IbvR_-I/KfpVyRzWQ0ASHdE8bXnx/biWdPWPymJp1dLczFR0k/D1x9DlretrpD_IbvR_-I -extld=clang $WORK/b001/_pkg_.a
/usr/local/go/pkg/tool/darwin_amd64/buildid -w $WORK/b001/exe/a.out # internal
```

The linker is not using `-s` and `-w` arguments, but others are persisted. So, Delve relies on DWARF and debug information inside a compiled binary, isn't it?

## Delve and `pkg/proc`

Surfing inside internal documentation of Delve you can find interesting parts of architecture, especially ["Notes on porting Delve to other architectures"](https://github.com/go-delve/delve/blob/master/Documentation/internal/portnotes.md#pkgproc):

> When porting Delve to a new CPU a new instance of the proc.Arch structure should be filled, see `pkg/proc/arch.go` and `pkg/proc/amd64_arch.go` as an example. To do this you will have to provide a mapping between DWARF register numbers and hardware registers in `pkg/dwarf/regnum` (see `pkg/dwarf/regnum/amd64.go` as an example). This mapping is not arbitrary it needs to be described in some standard document which should be linked to in the documentation of `pkg/dwarf/regnum`, for example the mapping for amd64 is described by the System V ABI AMD64 Architecture Processor Supplement v. 1.0 on page 61 figure 3.36.

The DWARF symbol table is an essential part of Delve, indeed. That's why we get the error using `go run main.go`.

## Conclusion

To avoid it in the future, make sure `go build` doesn't have `-ldflags` (you can get more details [here](https://pkg.go.dev/cmd/go#hdr-Compile_packages_and_dependencies)) argument with `-s` or `-w` flags (e.g. `-ldflags="-w"`) and don't use `go run main.go` for debugging.

Happy coding!