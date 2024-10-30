# Overview

Specs or specifications are type definitions.  Specs define the shape
of data using principles of an object oriented type system.  Specs
use inheritance for composition and may define arbitrary metadata.
There are two fundamental types of specs: *scalars* and *dicts*. Scalars
define an atomic type encoded as a string. Common scalars include Str,
Number, Date, Time, and DateTime. Dicts define a compound type composed
of zero or more *slots* that are the data fields.

# Names

Specs are always defined within a [lib](Libs.md).  User defined spec
names must obey the following restrictions:
  - Must contain only ASCII lower case letters, digits, or underbar
  - Must start with an ASCII upper case letter
  - Convention is to use camel case

The compiler may generate synthetic spec names which follow these restrictions
  - Start with underbar
  - Contain only ASCII digits
  - For example "_34" is a synthetic or auto name

We call the spec name within the library the *simple name*.  We combine
the simple name with the lib name using "::" to create the *qualified name*
or *qname*. Because lib names are globally unique, spec qnames are also
globally unique.  Examples:

```
  Str                         // simple name
  sys::Str                    // qualified name
  NaturalGasMeter             // simple name
  ph.equips::NaturalGasMeter  // qualified name
```

Dict slots are also specs that follow same naming restrictions but must start
with a lower case letter.  Slot specs have a qname formed from the parent
spec qname and a dot:

```
LibDepend: Dict {
  lib: Str                      // qname is sys::LibDepend.lib
  versions: LibDependVersions   // qname is sys::LibDepend.versions
}
```

# Syntax

Spec definitions follow the given syntax:

```xeto
SpecName: BaseType <meta1, meta2> {
  slotA: Type <slotMeta1, slotMeta2>
}
```

Scalar and list subtypes cannot define slots.  Here are some examples:

```xeto
// simple scalar
MyScalar: Scalar

// scalar with metadata
SocialSecurityNumber: Scalar <pattern:"\\d{3}-\\d{2}-\\d{4}">

// simple dict type
Point: Dict {
  x: Int
  y: Int
}

// dict with metadata
Person: Dict <sealed, icon:"user"> {
  name: Str
  height: Number <quantity:"length", minVal:0>
}
```

# Scalars

Scalar specs define an atomic type encoded as a string.  All scalars
inherit from `sys::Scalar`.  Scalars must define a canonical string
encoding used by instance data or in JSON.

Scalars may define a default value using the 'val' meta:

```xeto
// meta data
MyScalar: Scalar <val:"default">

// syntax sugar for same
MyScalar: Scalar "default"
```
A regular expression for their encoding using the 'pattern' meta:

```xeto
Date: Scalar <sealed, pattern:"\\d{4}-\\d{2}-\\d{2}"> "2000-01-01"
```

# Dicts

Dicts or dictionaries are specs that define a compound type.  Dicts
are equivalent to JSON object types.  Dicts are composed of *slots*.
Slots are named fields with an explicit value type.  We call the
containing spec of a slot the *parent*.  Slots are themselves specs
with a qname formatted as:

```
{parent qname}.{slot name}
```

Lets consider the example from above:

```xeto
Point: Dict {
  x: Int
  y: Int
}
```

This defines a spec named "Point" with two slots named "x" and "y".
The value space for the "x" and "y" fields is restricted to integers
via the `sys::Int` spec.

We can give slots a default value using the 'val' meta:

```xeto
Point: Dict {
  x: Int <val:"-99">
  y: Int <val:"-99">
}

// syntax sugar for same
Point: Dict {
  x: Int "-99"
  y: Int "-99"
}
```

# Lists

Specs may also be used to define a custom list type.  Typically
this is done via the 'of' meta to define the item type of the list:

```xeto
Numbers: List<of:Number>
```





