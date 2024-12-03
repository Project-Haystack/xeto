# Overview

Globals are slot spec definitions that globally apply to all dict specs
and instances.  They provide a way to force the use of a consistent slot
field type across all specs.  They enforce a [covariant](TypeSystem.md#covariance)
override in all specs of the parent lib and all libs that depend on
the parent lib. Additionally all instance data is restricted by the global
slots.

# Names

Globals follow same rules at [slot names](Specs.md#names):
  - Must contain only ASCII lower case letters, digits, or underbar
  - Must start with an ASCII lower case letter
  - Convention is to use camel case

# Example

Here is an example of a global spec:

```xeto
height: Number <quantity:"length", minVal:0>
```

This forces all dict specs in the parent lib and all libs that depend
on the parent lib to conform to this slot spec.  Examples dicts:

```xeto
// This spec is OK because it matches global type
SomeDict: Dict {
  height: Number
}

// This spec is OK because it is a covariant override
// compatible with the global since Int is a subtype of Number
AnotherDict: Dict {
  height: Int
}

// This will not compile because the type is not compatible
// with global; Str is not a subtype of Number
InvalidDict: Dict {
  height: Str
}
```

Globals also restrict instance data:

```xeto
// OK because slot matches global slot restrictions
@instance-1: { height: Number "12m" }

// Error: Global slot type is 'sys::Number', value type is 'sys::Date'
@instance-2: { height: Date "2024-12-03" }

// Error: Number must be 'length' unit; '°C' has quantity of 'temperature'
@instance-3: { height: Number 12°C }
```

# Inheritance

When a dict spec declares a slot that matches a global spec, then it
automatically inherits from that spec.  In the reflection APIs and AST
the slot will have a 'base' which references the global.  The dict
slot will inherit all the global's metadata.  In the example above, both
'SomeDict.height' and 'AnotherDict.height' will inherit the 'quantity'
and 'minVal' metadata tag from the global spec.

# Project Haystack Tags

Historically the Project Haystack ontology has required consistent use
of all tags across all entity types. The `ph` library uses globals to
enforce this design pattern. Every tag from 'ph' is defined as a global.
This requires that every lib that depends on 'ph' to declare slots
that are type compatible with the global tag definitions.

