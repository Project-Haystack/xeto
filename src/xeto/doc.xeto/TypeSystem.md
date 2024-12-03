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
The 'of' meta is used to parameterize container and reference types.
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

# Inheritance

TODO

# Covariance

TODO

