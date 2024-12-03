# Maybe

Slot specs may be annotated with the 'maybe' marker to make
them a *maybe type* or *option type*.  Maybe types mean that the
value may be omitted in instance data.  Maybe types are used
extensively, so we give then a special syntax - you can use
a question mark after the type name:

```xeto
// standard syntax for optional age field
age: Number <maybe>

// syntax sugar for above
age: Number?
```

When validating instance data, maybe slots can be omitted.  However, if
defined they must meet the slot spec's type and constraint requirements.

Xeto's covariance rules allow a maybe slot to be overridden as a non-maybe
type, but not vise versa.  For example the following is legal because
we are narrowing the value space in the subtype:

```xeto
OptionalAge: { age: Number? }

RequiredAge: OptionalAge { age: Number }
```

# Of
The `of` meta is used to parameterize container and reference types.
It is used with the following types:
  - `sys::List`: parameterize the item types
  - `sys::Ref`: parameterize the referent type
  - `sys::MultiRef`: parameterize the referent type
  - `sys::Query`: parameterize what the query returns

When validating covariance the 'of' type maybe narrowed, but it may not
be not widened.  For example the following is legal because we are narrowing
the list's 'of' type  to be more constrained and honoring the supertype's
contract:

```xeto
Foo: {
  list: List<of:Number>
}

Bar: Foo {
  list: List<of:Duration>
}
```

However, in this example we will get a compile time error because we
are widening the supertype's contract for list:

```xeto
Foo: {
  list: List<of:Number>
}

Bar: Foo {
  list: List<of:Obj>
}
```

We also use the 'of' type to perform [type inference](Specs.md#lists) for lists.

# Covariance

Covariance is the set rules for narrowing a spec definition.  Covariance
rules are checked when subtyping via inheritance or overriding a slot
from a supertype.  The primary idea of covariance is that we are
narrowing the possible set of values, but never widening it.  This ensures
the subtypes adhere to the type contract of their supertype.

Covariance requires the following sets of conditions:
- Slot type must be a subtype of the overridden slot
- `maybe` be can removed, but not added from base spec
- `of` meta must be a subtype of the base 'of' type
- `minVal` meta must be equal to or greater than base value
- `maxVal` meta must be equal to or less than base value
- `quantity` meta cannot be modified from base if defined
- `unit` meta cannot be modified from base if defined

Consider the spec below where slot 'a' has value space of the integers 2-7.
In the example, the subtype is covariant because it narrows the value space
to 3-5 (which adheres to the supertype's constraints):

```xeto
Foo: {
  a: Int <minVal:2, maxVal:7>
}

Bar: Foo {
  a: Int <minVal:3, maxVal:5>
}
```

However, this example is not covariant between the subtype is widening the
value space of the 'a' slot resulting in compile time error:

```xeto
Foo: {
  a: Int <minVal:2, maxVal:7>
}

Bar: Foo {
  a: Int <minVal:1, maxVal:8>
}
```
