[![Build Status](https://travis-ci.org/yugr/Implib.so.svg?branch=master)](https://travis-ci.org/yugr/Implib.so)

# Motivation

In a nutshell, Implib.so is a simple equivalent of [Windows DLL import libraries](http://www.digitalmars.com/ctg/implib.html) for POSIX shared libraries.

On Linux, if you link against shared library you normally use `-lxyz` compiler option which makes your application depend on `libxyz.so`. This would cause `libxyz.so` to be forcedly loaded at program startup (and its constructors to be executed) even if you never call any of its functions.

If you instead want to delay loading of `libxyz.so` (e.g. its unlikely to be used and you don't want to waste resources on it or you want to select best implementation at runtime), you can remove dependency from `LDFLAGS` and issue `dlopen` call manually. But this would cause `ld` to err because it won't be able to statically resolve symbols which are supposed to come from this shared library. At this point you have only two choices:
* emit normal calls to library functions and suppress link errors from `ld` via `-Wl,-z,nodefs`; this is undesired because you loose ability to detect link errors for other libraries statically
* load necessary function addresses at runtime via `dlsym` and call them via function pointers; this isn't very convenient because you have to keep track which symbols your program uses and also somehow manage global function pointers

Implib.so provides an easy solution - link your program with a _wrapper_ which
* provides all necessary symbols to make linker happy
* loads wrapped library on first call to any of its functions
* redirects calls to library symbols
Generated wrapper code is analogous to Windows import libraries which achieve the same functionality for DLLs.

Implib.so can also be used to [reduce API provided by existing shared library](#reducing-external-interface-of-closed-source-library) or [rename it's exported symbols](#renaming-exported-interface-of-closed-source-library).

Implib.so was originally inspired by Stackoverflow question [Is there an elegant way to avoid dlsym when using dlopen in C?](https://stackoverflow.com/questions/45917816/is-there-an-elegant-way-to-avoid-dlsym-when-using-dlopen-in-c/47221180).

# Usage

A typical use-case would look like this:

```
$ implib-gen.py libxyz.so
```

This will generate code for host platform (presumably x86\_64). For other targets do

```
# TARGET can be arm-linux-gnueabi, aarch64-linux-gnu, x86_64-linux-gnu
$ implib-gen.py --target $TARGET libxyz.so
```

Script generates two files: `libxyz.so.tramp.S` and `libxyz.so.init.c` which need to be linked to your application (instead of `-lxyz`):

```
$ gcc myfile1.c myfile2.c ... libxyz.so.tramp.S libxyz.so.init.c ... -ldl
```

(note that you need to link against libdl.so).

Application can then freely call functions from `libxyz.so` _without linking to it_. Library will be loaded (via `dlopen`) on first call to any of its functions. If you want to forcedly resolve all symbols (e.g. if you want to avoid delays further on) you can call `void libxyz_init_all()`.

Above command would perform a _lazy load_ i.e. load library on first call to one of it's symbols. If you want to load it at startup, run

```
$ implib-gen.py --no-lazy-load libxyz.so
```

If you don't want `dlopen` to be called automatically and prefer to load library yourself at program startup, run script as

```
$ implib-gen.py --no-dlopen libxys.so
```

If you do want to load library via `dlopen` but would prefer to call it yourself (e.g. with custom parameters or with modified library name), run script as

```
$ implib-gen.py --dlopen-callback=mycallback libxyz.so
```

(callback must have signature `void *(*)(const char *lib_name)` and return handle of loaded library).

Finally to force library load and resolution of all symbols, call

    void _LIBNAME_tramp_resolve_all(void);

# Reducing external interface of closed-source library

_TODO: modifying visibility of symbols .dynsym might be easier..._

Sometimes you may want to reduce public interface of existing shared library (e.g. if it's a third-party lib which erroneously exports too many unrelated symbols).

To achieve this you can generate a wrapper with limited number of symbols and override the callback which loads the library to use `dlmopen` instead of `dlopen` (and thus does not pollute the global namespace):

```
$ cat mysymbols.txt
foo
bar
$ cat mycallback.c
#define _GNU_SOURCE
#include <dlfcn.h>
#include <stdio.h>
#include <stdlib.h>

#ifdef __cplusplus
extern "C"
#endif
void *mycallback() {
  void *h = dlmopen(LM_ID_NEWLM, "libxyz.so", RTLD_LAZY | RTLD_DEEPBIND);
  if (h)
    return h;
  fprintf(stderr, "dlmopen failed: %s\n", dlerror());
  exit(1);
}

$ implib-gen.py --dlopen-callback=mycallback --symbol-list=mysymbols.txt libxyz.so
$ ... # Link your app with libxyz.tramp.S, libxyz.init.c and mycallback.c
```

# Renaming exported interface of closed-source library

Sometimes you may need to rename API of existing shared library to avoid name clashes.

To achieve this you can generate a wrapper with _renamed_ symbols which call to old, non-renamed symbols in original library loaded via `dlmopen` instead of `dlopen` (to avoid polluting global namespace):

```
$ cat mycallback.c
... Same as before ...
$ implib-gen.py --dlopen-callback=mycallback --symbol_prefix=MYPREFIX_ libxyz.so
$ ... # Link your app with libxyz.tramp.S, libxyz.init.c and mycallback.c
```

# Overhead

Implib.so overhead on a fast path boils down to
* predictable direct jump to wrapper
* predictable untaken direct branch to initialization code
* load from trampoline table
* predictable indirect jump to real function

This is very similar to normal shlib call:
* predictable direct jump to PLT stub
* load from GOT
* predictable indirect jump to real function

so it should have equivalent performance.

# Limitations

The tool does not transparently support all features of POSIX shared libraries. In particular
* it can not provide wrappers for data symbols
* it makes first call to wrapped functions asynch signal unsafe (as it will call `dlopen` and library constructors)
* it may change semantics if there are multiple definitions of same symbol in different loaded shared objects (runtime symbol interposition is considered a bad practice though)
* it may change semantics because shared library constructors are delayed until when library is loaded

Also note that the tool is meant to be a PoC. In particular I didn't implement the following very important features:
* proper support for multi-threading
* symbol versions are not handled at all
* support i386 and OSX

None of these should be hard to add so let me know if you need it.

Finally tool is only lightly tested and minor TODOs are scattered all over the code.

# Related work

As mentioned in introduction import libraries are first class citizens on Windows platform:
* [Wikipedia on Windows Import Libraries](https://en.wikipedia.org/wiki/Dynamic-link_library#Import_libraries)
* [MSDN on Linker Support for Delay-Loaded DLLs](https://msdn.microsoft.com/en-us/library/151kt790.aspx)

Lazy loading is supported by Solaris shared libraries but was never implemented in Linux. There have been [some discussions](https://www.sourceware.org/ml/libc-help/2013-02/msg00017.html) in libc-alpha but no patches were posted.

Implib.so-like functionality is used in [OpenGL loading libraries](https://www.khronos.org/opengl/wiki/OpenGL_Loading_Library) e.g. [GLEW](http://glew.sourceforge.net/) via custom project-specific scripts.
