# Overview

[Haxall](https://haxall.io) is an open source framework that includes
the reference implementation of Xeto. It provides a suite of command
line tools for working with Xeto.  Haxall is written in the [Fantom](https://fantom.org)
programming language which allows the Xeto tools to run with either
NodeJS or Java.

# Install for Java

Follow the setup instructions from [Haxall docs](https://haxall.io/doc/docHaxall/Setup):

1. Install Java on your system
2. Download and unzip the last release from [GitHub](https://github.com/haxall/haxall/releases)
3. Run 'bin/xeto version' to verify install

# Install for NodeJS

Install [@haxall/haxall](https://www.npmjs.com/package/@haxall/haxall) using
npm as follows:

1. Install using 'npm i -g @haxall/haxall' (if you get a permission error on Unix
   then run via sudo)
2. Run 'xeto version' to verify install

See NPM docs on package.json [bin scripts](https://docs.npmjs.com/cli/v10/configuring-npm/package-json#bin) for additional details.

# Env Path

The Haxall tools all rely on a search path used to find Xeto libraries.  The
path is composed of a list of one or more directories on the local
file system which is searched from beginning to end.  The first directory
is called the *workDir* and it is the default location to create and
install new libs.  The last directory in the path is called the *homeDir*
and is always where you installed Haxall.

The workDir and path are configured based on your current working
directory by walking up the path looking for one of the following files
to mark the root of your workDir:

1. 'xeto.props' - explicit configuration for Xeto
2. 'fan.props' - use standard [Fantom path](https://fantom.org/doc/docLang/Env#PathEnv) behavior
3. '.git' - assume you are developing new libs with Git version control
4. Fallback is to assume workDir is same as homeDir (where you installed Haxall)

You can configure additional directories in your search path by adding the 'path'
line to your "xeto.props" file. The value is a list of directories separated by
a semicolon (note we do not use colon even on Unix).  You can use "../" to
specify a directory relative to the workDir.

Example "xeto.props":

```
path=/work/project-libs;/work/global-libs
```

You can debug your environment setup using the 'xeto version' command:

```
/work/acme-lib> xeto version

...

xeto.version:    3.1.12
xeto.mode:       xeto.props
xeto.workDir:    /work/acme-libs
xeto.homeDir:    /work/haxall-home
xeto.installDir: /work/acme-libs
xeto.path:
  /work/acme-libs
  /work/project-libs
  /work/global-libs
  /work/haxall-home
```

We always search the path looking for source and xetolib files according
to the naming convention:

```
{dir}/src/xeto/{lib-name}/
{dir}/lib/xeto/{lib-name}/
```

So in the above example we would search the following directories
for a library named "acme.templates":

```
/work/acme-libs/src/xeto/acme.templates/
/work/acme-libs/lib/xeto/acme.templates/
/work/project-libs/src/xeto/acme.templates/
/work/project-libs/lib/xeto/acme.templates/
/work/global-libs/src/xeto/acme.templates/
/work/global-libs/lib/xeto/acme.templates/
/work/haxall-home/src/xeto/acme.templates/
/work/haxall-home/lib/xeto/acme.templates/
```





