# Overview

Specs or specifications are type definitions.  Specs define the shape
of data using principles of an object oriented type system.  Specs
use inheritance for composition and may define arbitrary metadata.
There are two fundamental types of specs: *scalars* and *dicts*. 
  - [**Scalars**](https://github.com/Project-Haystack/xeto/blob/master/src/xeto/doc.xeto/Specs.md#scalars) define an atomic type encoded as a string. Common scalars include Str,
Number, Date, Time, and DateTime.
  - [**Dicts**](https://github.com/Project-Haystack/xeto/blob/master/src/xeto/doc.xeto/Specs.md#dicts) define a compound type composed of zero or more *slots* that are the data fields.

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
Scalars may define a regular expression for their encoding
using the 'pattern' meta:

```xeto
Date: Scalar <sealed, pattern:"\\d{4}-\\d{2}-\\d{2}"> "2000-01-01"
```

# Dicts

Dicts or dictionaries are specs that define a compound type.  Dicts
are equivalent to JSON object types.  Dicts are composed of *slots*.
Slots are named fields with an explicit value type.

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

# Slots

Slot specs are specs nested inside a dict spec to define a field
value in instance data.  They follow all the same rules as other specs,
but must start with a lower case ASCII letter.

We call the containing spec of a slot the *parent*.  Slots have
a qname formatted as:

```
{parent qname}.{slot name}
```

# Lists

The `sys::List` type is used to define an ordered list of values.
The list item type may be parameterized by the 'of' meta:

```xeto
Foo: {
  numbers: List <of:Number>  // List that contains only Numbers
}
```

In Xeto instance data that contains a list uses curly braces (we do
not use '[]' like JSON):

```xeto
// explicitly typed list with individual items typed
@foo-1: Foo {
  numbers: List <of:Number> { Number 2, Number 3, Number 4}
}

// items are implicitly typed as Number from the 'of'
@foo-2: Foo {
  numbers: List <of:Number> { 2, 3, 4}
}

// typed list is inferred from Foo.numbers slot definition
@foo-3: Foo {
  numbers: { 2, 3, 4}
}
```

The `sys::List` type is sealed, you cannot derive new types
from it via inheritance:

```xeto
// This will not compile
MyNumbersList: List <of:Number>
```

# Representation

Specs are modeled as dict data just like [instance data](Instances.md).
The following tags are defined by Xeto itself and may not be used
as metadata tags:

  - 'id': a Ref which is the spec qname
  - 'base': a qname Ref to the type inherited from; null for `sys::Obj`;
  - 'type': a qname Ref to slot type
  - 'parent': a qname Ref to the parent dict if spec is a slot
  - 'doc': documentation from comments
  - 'spec': always a Ref to `sys::Spec` (the spec is an instance of Spec)
  - 'slots': the slot specs of a dict spec

Type specs use 'base' and slot specs use 'type'.

Note: the compiler currently reserves several other metadata tags for
future proofing.

When representing a spec as a dict or JSON object, the tags above are
merged with the spec meta.  Example:

```xeto
// Xeto spec example
Person: Dict <sealed, icon:"user"> {
  name: Str  // Full name
}
```

Haystack representation of the example spec:

```
id: @acme::Person
spec: @sys::Spec
base: @sys::Dict
doc: "Xeto spec example"
sealed
icon: "user"
slots: {
  name: {
    id: @acme::Person.name
    spec: @sys::Spec,
    type: @sys::Str,
    doc: "Full name"
  }
}
```

JSON representation of the example spec:

```json
"Person": {
  "id": "acme::Person",
  "spec": "sys::Spec",
  "base": "sys::Dict",
  "sealed": "✓",
  "icon": "user",
  "slots": {
    "name": {
      "id": "acme::Person.name",
      "spec": "sys::Spec",
      "type": "sys::Str",
      "doc": "Full name"
    }
  }
}
```

Note that when mapping specs to Haystack or JSON there is a level
of type erasure as discussed in [Fidelity](Fidelity.md).

