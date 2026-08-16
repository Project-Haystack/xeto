# Overview

This chapter covers topics for building a lib:

- [build vars](#build-vars): share metadata across project
- [packaging](#packaging): select resource files to bundle into the xetolib

# Build Vars

It is common to define many Xeto libs that all share the same metadata such
as version, dependency constraints, or org meta.  You can define this
metadata once and reference it using *build variables*.

Build variables are defined in a file named `src/xeto/xeto-build.props` in the
root of your source environment.  It is formatted as a [props file](Grammar.md#props-file).

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

Names prefixed with `xeto.` are reserved for the compiler.  Unlike the
variables above they are never referenced by a `BuildVar` placeholder;
they configure how the lib is built.  No reserved props are currently
defined.

# Packaging

Not every file beside your source belongs in the shipped library.  Build
scripts, scratch data, and the output of other toolchains all live in a
source tree without being part of what the lib distributes.

Packaging is therefore *opt-in*: the lib always carries its intrinsic
content, and everything else is selected by the author.  Intrinsic content
is owned by the compiler: xeto source, chapters, and "xeto-" system files.

Resource files are included via the following lib pragma fields:

  - `include`: packaged into the lib, but with no uri of its own.  Use it
    for files your code reads at runtime.
  - `publish`: packaged *and* given its own uri so it can be linked and
    fetched individually.

Selecting a file with `publish` implies `include`, so a file is never
listed twice.  A file selected by neither is not packaged at all.

```xeto
pragma: Lib <
  version: "1.0.0"
  depends: { { lib: "sys" } }
  include: {"/data"}
  publish: {"/doc", "/logo.svg", "/img/*.svg"}
>
```

Both tags take a list of [sys::LibFilePattern].  A pattern is written the
same way a lib file is addressed - rooted at the lib with a leading slash.
Each pattern takes one of three forms:

```
/doc              // directory: every file in the subtree
/doc/index.md     // file: exactly one file
/*.svg            // extension: the svg files at the lib root
/doc/*.svg        // extension: the svg files directly in "/doc"
```

A directory carries its whole subtree, while the `"*.ext"` form selects only
the files sitting directly in the named directory.  The `"*"` wildcard may
appear only as the whole final section of the path.  Patterns have no trailing
slash, no "..", and no backslashes.

A pattern which selects no file is a build error rather than a silent
no-op, so a renamed or deleted file is caught at compile time.

Because a published file takes a uri of its own, its name must obey the
[file name](Libs.md#file-names) restrictions.  An included file never
takes a uri, so it may be named whatever the toolchain which produced it
happens to call it.

