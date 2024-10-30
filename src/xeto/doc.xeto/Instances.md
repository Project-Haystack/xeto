# Overview

Instances are the objects the represent the data modeled
by specs. Instances are always dicts (dictionaries) or
JSON objects.  Instances may be defined within a [lib](Libs.md)
or outside of a lib.

# Id
All instances must define an 'id' tag that is the unique identifier
for the data entity.  The 'id' tag must always a Ref value.  Instances
which are defined within a lib will have a id formed from the qname:

```xeto
// instance in lib acme.widgets; id is @acme.widgets::sku-123
@sku-123: Part {...}
```

Instances outside of a lib will have an identifier that is
project or system based.  It is illegal for a non-lib id
to contain "::" double colons; that format is reserved
for lib instances.

The scope of uniqueness for lib instances is global due to the
fact that lib names are [globally unique](Libs.md#names).  For
non-lib instances the scope of uniqueness is only guaranteed
with the instance's dataset.

# Spec

Instances declare their type spec via the 'spec' tag.  The value
is a Ref with the spec qname.  When the instance is declared via
Xeto in a lib, the 'spec' tag is implied:

```
// Xeto instance - spec is ph::ElecMeter via name resolution
@meter-1: ElecMeter {...}

// Haystack instance
id: @meter-1
spec: @ph::ElecMeter

// JSON object
{
  "id": "meter-1",
  "spec": "ph::ElecMeter"
}
```

# Syntax

Lib instances follow the given syntax:

```xeto
@id: TypeSpec {
  slot1
  slot2
  ...
}
```

Unlike specs, instances cannot define meta using '<>'.

For example to create an instance of a floor and room:

```xeto
@floor-2: Floor {
  dis: "Floor 2"
}

@room-204: Room {
  dis: "Room 204"
  spaceRef: @floor-2
}
```

Note that we can cross reference other instances inside the lib
using relative ids.  All the non-maybe tags from the spec type
are automatically added into the instance.  The instances in
the example above evaluate to the following Haystack dicts:

```trio
id: @acme::floor-2
spec: @ph::Floor
dis: "Floor 2"
space
floor
---
id: @acme::room-204
spec: @ph::Room
spaceRef: @acme::floor-2
space
room
```

Note that the `space` and `floor`/`room` markers are implied
from the instance's spec.

The instances above map the following JSON representation:

```json
{
  "id": "acme::floor-2"
  "spec": "ph::Floor",
  "dis": "Floor 2",
  "space": "✓",
  "floor": "✓"
}

{
  "id": "acme::room-204"
  "spec": "ph::Room",
  "dis": "Room 204",
  "spaceRef": "acme::floor-2",
  "space": "✓",
  "room": "✓"
}
```

# Nesting Instances

Xeto allows instances to be nested into a tree of dicts.
Nested instances can have a top-level id, or a slot name,
or both.

Here is an example where the nested dicts do not have
a top-level id:

```xeto
@toolbar: Toolbar {
  save: Button { text:"Save" }
  exit: Button { text:"Exit" }
}
```

This maps to the following Haystack dict where 'save' and 'exit'
are just named slots that contain a nested dict:

```trio
id:@acme::toolbar
exit:{text:"Exit" spec:@acme::Button}
save:{text:"Save" spec:@acme::Button}
spec:@acme::Toolbar
```

But we can also make those nested dicts first class instances:

TODO


