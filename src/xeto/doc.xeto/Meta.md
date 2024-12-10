# Overview

Libs and specs can define arbitrary *meta* or metadata.  Many of xeto's
built-in features leverage meta to annotate specs.  For example, the `sealed`
meta is used by the compiler to indicate that a spec cannot be subtyped.
Both lib and spec meta is a normal dict, but uses special rules for
static  typing.  Meta tags are formally defined using a specialized
[global spec](Globals.md).  These *meta spec*s define a single tag that may be
used in meta dicts.  It is required that all meta tags are defined by a
meta spec in the library's [namespace](Namespaces.md#lib-namespaces).

# Names

Meta specs follow same rules at [globals](Globals.md) and
[slot names](Specs.md#names):
  - Must contain only ASCII lower case letters, digits, or underbar
  - Must start with an ASCII lower case letter
  - Convention is to use camel case

There are additional restrictions on meta names that are reserved
for the [AST representation](Specs.md#representation).  The following
meta names are reserved:

```
base, class, data, def, id, is, lib, loaded, loc, instances, name,
parent, qname, slot, slots, spec, specs, super, supers, type, types,
and any name starting with "xeto"
```

Additionally, it is invalid to declare a meta spec that is already
define by 'sys' or in any other dependent lib.

# Syntax

Meta specs follow the same syntax rules as [global specs](Globals.md#syntax)
except they include the `meta` metadata tag:

```xeto
metaTag: Type <meta, moreMeta>
```

Here are concrete examples from the 'sys' lib:

```xeto
// Abstract types cannot be implemented directly by instance data
abstract: Marker <meta, noInherit>

// Applied to strings and lists to specify inclusive maximum length
maxSize: Int <meta>
```
