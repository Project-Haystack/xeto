# Overview

Xeto uses a form of object oriented inheritance as the main
mechanism for composition.  All type specs are declared to inherit
from one or more other specs.  The only exception to this rule
is `sys::Obj` which is the root of the type hierarchy (it is the
top of any type).

Inheritance is used semantically to define a taxonomy of concepts.  We
use the terms *supertype* and *subtype* to define the two sides of an
inheritance relationship.  For example, `ph::Water` is a subtype `ph::Liquid`;
conversely we say that `ph::Liquid` is the supertype `ph::Water`. From
a semantic or type theory perspective this means that water is a specific
type of liquid but that there are other types of liquid that are not water
(such as `ph::Gasoline`).

Inheritance is also used structurally to have subtypes inherit metadata
and slots from supertypes. Inheritance rules ensure the Liskov Substitution
Principle which guarantees a subtype is substitutable for its supertype.
To enforce these rules, the Xeto compiler includes many compile time checks
found in statically typed programming language.

# Meta

Specs inherit most metadata from their supertype(s).  Meta is not
inherited if it is defined with the `noInherit` tag (such as the `abstract`
meta tag).  Subtypes may add additional meta tags and may override
meta from the supertype only if [covariant](TypeSystem.md#covariance).

# Slots

Dict subtypes inherit all the slots from their supertype(s).  Subtypes
can add new slots with new names.  And subtypes can override slots
from their supertype as long as they follow [covariance](TypeSystem.md#covariance)
rules.

In the example below the spec 'Bar' implicitly inherits the slots 'a' and
'b', plus adds its own slot 'c':

```xeto
Foo: {
  a: Str
  b: Date
}

Bar: Foo {
  c: Number
}
```

In this example, we show how a subtype can covariantly override a slot
to narrow the type and/or add additional meta:

```xeto
Foo: {
  num: Number
}

Bar: Foo {
  num: Int <minVal:0>
}
```



