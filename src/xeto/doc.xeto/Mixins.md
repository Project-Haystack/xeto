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
# Extended Spec

The extended spec is computed as follows:
- search the namespace for libs that register a mixin
- merge in the meta tags on the spec
- merge in the meta tags on each slot
- merge in new extended slots
- it is invalid to add a global in a mixin

If there is a naming collision for spec/slot meta, then we
use the one from the library lower in the dependency chain.

If there is a naming collision for an extended slot, then
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


