# Overview

Libs or libraries are used to package Xeto into versioned modules.  Libs
are deployed as zip files with the "xetolib" extension and contain the
source for the library's specs, instances, and additional resource
files.  Libs define explicit dependencies for the specs and instances they
import into their namespace.

# Names

Libs have a globally unique dotted name.  Lib names must obey the following
restrictions:
  - Must contain only ASCII lower case letters, digits, or underbar
  - Use dot as separator character
  - Each dotted section must start with a lowercase ASCII letter
  - Dots and underbars may only be used between letters and digits

Example of valid vs invalid lib names:

```
foo         // ok
foo.bar     // ok
foo.bar_13  // ok
foo!bar     // error - invalid char "!"
foo.7bar    // error - dotted sections must start with lowercase
foo_.bar    // error - underbar may not be adjacent to dot
foo__bar    // error - underbar may not be adjacent to another underbar
foo.barBaz  // error - only lower case ASCII letters permitted
```

The lib name is determined by the directory name of the Xeto source
code.  For example to create library called "acme.assets", then your
source tree would be organized:

```
  src/xeto/
    acme.assets/      // directory name determines lib name
      lib.xeto        // must have a pragma file with this name
      specs.xeto      // additional definitions
```

# Lib Name Prefixes

In order to guarantee globally unique names, the following conventions are
used for lib name prefixes:

- Reverse DNS names are reserved for the top level domains: com, org, net,
  gov, edu, and io
- The prefix "cc." is reserved for community contributions
- Top level prefixes are granted when appropriate to avoid name squatting

The Project Haystack non-profit organization manages prefix registration
on the https://xeto.dev website.  Both "cc." and top level prefixes should
be registered on the website for your organization.

For example, if you own the DNS domain "acme.com", then you automatically
own the "com.acme" lib name (as well as names that start with "com.acme.").
This same convention is also followed by the Java community for package naming.
Note: there is a plethora of TLDs now, we currently only standardize a
limited set (the most common ones).

If you are vendor or have a project with a name such as FooBar, then
you can request a top level prefix for "foobar" at https://xeto.dev.  Or if
developing a community contribution for a vendor named "Baz", you
can request "cc.baz".  Top-level prefixes that would conflict with
DNS top level domains such as country codes will not be granted.

# Pragma

All libs must define their metadata file in a file named "lib.xeto".
Lib meta includes:
  - version
  - dependencies
  - summary documentation
  - organization metadata

Lib metadata is declared in lib.xeto via the *pragma*.  Here is an
example template:

```xeto
pragma: Lib <
  doc: "Summary of the lib goes here"
  version: "2.0.6"
  depends: {
    { lib: "sys", versions: "1.0.x" }
    { lib: "foo.bar", versions "2.x.x" }
  }
  org: {
    dis: "Acme, Inc"
    uri: "https://acme.com/"
  }
>
```

Specific lib metadata is discussed in detail below.

## Doc

The pragma 'doc' tag declares a short summary of the library.  This tag should
be used for summary information only, not the complete documentation.  The
first sentence will be used for the lib in documentation indexes.

## Version

The pragma 'version' tag declares the library version.  This must be
string formatted as three decimal digits separated by a dot.

## Depends

The pragma 'depends' tag specifies a list of lib dependencies that are
formatted as lib name and version constraints.  Depends are used
to define the dependency graph of a lib which imports all spec and
instance names from the dependencies in to the library's namespace.

Version constraints are formatted a string with the following BNF grammar

```
<range>    :=  <ver> "-" <ver>
<ver>      :=  <seg> "." <seg> "." <seg>
<seg>      :=  <wildcard> | <number>
<wildcard> :=  "x"
<number>   :=  <digit>+
<digit>    :=  "0" - "9"
```

Examples for version constraints:

```
1.2.3         // version 1.2.3 exact
1.2.x         // any version that starts with "1.2."
3.x.x         // any version that starts with "3."
x.x.x         // any version - wildcard match
1.0.0-2.0.0   // range from 1.0.0 to 2.0.0 inclusive
1.2.0-1.3.x   // range from 1.2.0 to 1.3.* inclusive
```

If a dependency's 'versions' tag is omitted, then it is assumed
to be "x.x.x" (any version).

## Org

The pragma 'org' tag specifies summary information for the organization
publishing the library. Org is a dict with the following tags:
  - 'dis': display name for the organization
  - 'uri': URL to the organization's web site

