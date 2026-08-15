# Overview

It is common to define many Xeto libs that all share the same metadata such
as version, dependency constraints, or org meta.  You can define this
metadata once and reference it using *build variables*.

Build variables are defined in a file named `src/xeto/build.props` in the
root of your source environment.  It is formatted as a [props file](Grammar.md#props-file).
The same file also carries a small number of reserved variables which configure
the compiler itself; see [Reserved Vars](#reserved-vars).

# Defining Vars

Variable names must use your lib prefix. Build variables are inherited when
using [pathing](doc.xeto.tools::Setup#env-path).

Here is an example file:

```
ph.version=5.0.0
ph.depend=5.0.0
ph.license=AFL-3.0
ph.org.dis=Project Haystack
ph.org.uri=https://project-haystack.org/
```

Then in your `lib.xeto` you can reference these variables as placeholders
via the [sys::BuildVar] scalar type:

```xeto
pragma: Lib <
  doc: "Project haystack points library"
  version: BuildVar "ph.version"
  depends: {
    { lib: "sys", versions: BuildVar "ph.depend" }
    { lib: "ph",  versions: BuildVar "ph.depend" }
  }
  categories: {"ph"}
  license: BuildVar "ph.license"
  org: {
   dis: BuildVar "ph.org.dis"
   uri: BuildVar "ph.org.uri"
  }
>
```

Variables actually referenced by a `BuildVar` placeholder are captured into
the [xetolib](Libs.md#xetolib) so the lib can be recompiled later with the
metadata it was originally built with.

# Reserved Vars

Names prefixed with `xeto.` are reserved for the compiler.  Unlike the
variables above they are never referenced by a `BuildVar` placeholder;
they configure how the lib is built.

# Excluding Source Dirs

Sometimes a lib source directory contains files used to *build* the lib
which should not be packaged into it.  A common case is a directory of
sources compiled by another toolchain into a directory the lib does ship:
you package the generated output, not the inputs it was generated from.

Use the reserved `xeto.srcExclude` variable to omit directories from every
lib in the project:

```
xeto.srcExclude=src/;ignore/
```

The value is a semicolon separated list and each entry must end with `/`.
Entries name a directory at the root of a lib, so `src/` excludes all
files under `{lib}/src/` but not `{lib}/js/src/`.

Excluded directories are skipped entirely: their files are not compiled,
not packaged into the xetolib, and not exposed as lib files.  Because
their names never map to a URI, they are also exempt from the
[file name](Libs.md#file-names) restrictions - useful since external
toolchains often generate names Xeto would otherwise reject.

