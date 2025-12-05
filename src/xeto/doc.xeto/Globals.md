# Overview

Globals are spec definitions that globally apply to all types and
instances under a given type.  They provide a way to force the use
of a consistent [covariant](TypeSystem.md#covariance) slot type across
all specs of the parent type. Additionally all instance data inheriting
from the parent type is restricted by global definitions.

We use the term *member* to describe a slot or global defined with
a given type spec.  Slots are used to declare that a type has an expected
field.  Globals are used to ensure consistency of a given field across
different types.  A good example is the 'PhEntity.area' global that
requires all usage of the 'area' slot to be a Number type with an area
unit. Not all PhEntity's have an area, but when they do such as 'Site'
and 'Space' that tag must be used consistently.

# Names

Globals follow same rules as [slot names](Specs.md#names):
  - Must contain only ASCII lower case letters, digits, or underbar
  - Must start with an ASCII lower case letter
  - Convention is to use camel case

Additionally, it is invalid to declare a global that is already
defined by the parent type.

# Syntax

Global members look just like  [slot specs](Specs.md#slots)
with a leading "*" asterisk:

```xeto
MyType: {
  *globalTag: Type <meta1, meta2> "default"
}
```

The asterisk is syntax sugar for adding the `<global>` meta tag:

```xeto
MyType: {
  globalTag: Type <global, meta1, meta2> "default"
}
```

# Examples

Here is an example of a global spec:

```xeto
Person: {
  *height: Number <quantity:"length", minVal:0>
}
```

This forces all specs that subtype from Person to conform to this
global spec.  Example subtypes:

```xeto
// This spec is OK because it matches global type
SomePerson: Person {
  height: Number
}

// This spec is OK because it is a covariant override
// compatible with the global since Int is a subtype of Number
AnotherPerson: Person {
  height: Int
}

// This will not compile because the slot type is not
// compatible with the global; Str is not a subtype of Number
InvalidPerson: Person {
  height: Str
}
```

Globals also restrict instance data:

```xeto
// OK because slot matches global slot restrictions
@instance-1: Person { height: Number "12m" }

// Error: Global slot type is 'sys::Number', value type is 'sys::Date'
@instance-2: Person { height: Date "2024-12-03" }

// Error: Number must be 'length' unit; '°C' has quantity of 'temperature'
@instance-3: Person { height: Number 12°C }
```

# Inheritance

When a dict spec declares a slot that overrides a global, then it
automatically inherits from that spec.  In the reflection APIs and AST
the slot will have a 'base' which references the global.  The dict
slot will inherit all the global's metadata.  In the example above, both
'SomePerson.height' and 'AnotherPerson.height' will inherit
the 'quantity' and 'minVal' metadata tag from the global spec.

# Representation

Global members are modeled just like any slot with the 'global'
meta tag.

# Project Haystack Tags

Historically the Project Haystack ontology has required consistent use
of all tags across all entity types. The `ph` library uses globals to
enforce this design pattern. Every tag from 'ph' is defined as a global
in the base 'PhEntity' type:

```xeto
PhEntity: Entity {
  // Area of a shape or floor space
  *area: Number <quantity:"area">

  // Equipment used to store electric energy
  *battery: Marker

  ...
}
```