# Overview

Mixins are specialized specs used to extend the definitions of existing
specs.  A mixin is used to add meta and slots to a spec from a dependent
library via late binding.  Within a given namespace the *extended spec* is
the representation of the original spec merged with all the mixins registered
by the namespace's libs.

# Syntax

Consider the following original spec:

```xeto
Person: Dict <abstract> {
  dis: Str
  age: Number?
}
```

A mixin in another library can add metadata to the Person spec
like this:

```xeto
+Person <icon:"user">
```

You can add metadata to the Person slots as follows (note you
do not repeat the slot type):

```xeto
+Person {
  age: <icon:"calendar">
}
```

You can add an extended slot as follows:

```xeto
+Person {
  orgRef: Ref <of:Org>
}
```

Or you can use any combination to add spec meta, slot meta, or
new slots:

```xeto
+Person <icon:"user"> {
  age: <icon:"calendar">
  orgRef: Ref <of:Org>
}
```

# Globals

A mixin may also declare [globals](Globals.md) to contribute tag
typing for the extended type's chain:

```xeto
+PhEntity {
  // BACnet protocol addressing
  *bacnetAddr: BacnetAddr
}
```

A mixin slot adds a structural member to the extended type; a mixin
global only lays down the type for a tag name.  Libraries that
declare the mixin's library as a dependency resolve their declared
slots against mixin members - both slots and globals - exactly like
inherited members: an untyped slot infers its type, a typed slot is
covariance checked, and the slot's 'base' references the mixin
member.  Resolution scope is the declared dependency chain, so the
same source with the same dependencies always compiles the same way
regardless of what other libs the namespace contains.

Two mixins may contribute the same member name; that is legal and
the extended spec surfaces both.  However a slot declaring that name
cannot inherit against it - that is an ambiguous inheritance error.

# Multi-File Mixins

A mixin may be defined by more than one `+` block in the same library,
including blocks in different source files.  The slots are merged into a
single effective mixin.  This lets you organize a large mixin (a `Funcs`
library, for example) across several files:

```xeto
// file-a.xeto
+Person {
  orgRef: Ref <of:Org>
}

// file-b.xeto
+Person {
  managerRef: Ref <of:Person>
}
```

The result is the same as if both slots were declared in one block.

Only one of the blocks may carry mixin meta; the rest must be slots-only.
It is an error for two blocks of the same mixin to declare meta.

```xeto
// the primary block carries the mixin's meta
+Person <icon:"user"> {
  orgRef: Ref <of:Org>
}

// additional blocks are slots-only
+Person {
  managerRef: Ref <of:Person>
}
```

Note this features applies *only* to mixins (the `+` form).  A normal type
definition may not be split: declaring the same `Foo:` type in two places
is still a duplicate error.

# Extended Spec

The extended spec is computed as follows:
- search the namespace for libs that register a mixin
- merge in the meta tags on the spec
- merge in the meta tags on each slot
- merge in new extended slots
- merge in the mixin's globals

If there is a naming collision for spec/slot meta, then we
use the one from the library lower in the dependency chain.

If there is a naming collision for an extended slot or global, then
the runtime should keep track of the duplicate names.  This
allows the application level to detect ambiguous unqualified
names and implement application specific behavior.

In the example above the extended spec representation is:

```xeto
Person <abstract, icon:"user"> {
  dis: Str
  age: Number? <icon:"calendar">
  orgRef: Ref <of:Org>
}
```

Note that the extended spec definition is completely based on the
selected libs in the namespace.  It might be different in other
namespaces with alternate libs.  This is why we say the extended
spec is computed at runtime.

# Representation

The AST representation for a mixin looks exactly like any other
spec with the following exceptions:
- the mixin has the 'mixin' meta tag
- the mixin uses the 'base' meta tag to reference the original spec

