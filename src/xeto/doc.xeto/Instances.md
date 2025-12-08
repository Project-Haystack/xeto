# Overview

Instances are the objects that represent the data modeled
by specs. Instances are always dicts (dictionaries) or
JSON objects.  Instances may be defined within a [lib](Libs.md)
or outside of a lib.

# Id
All instances must define an 'id' tag that is the unique identifier
for the data entity.  The 'id' tag must always be a Ref value and cannot
start with an uppercase letter.  Instances which are defined within a
lib will have a id formed from the qname:

```xeto
// instance in lib acme.widgets; id is @acme.widgets::sku-123
@sku-123: Part {...}
```

Instances defined within a lib must not start with a uppercase letter.
This allows us to infer from a qname id whether it is a spec or
an instance.  Also it invalid to have a lib instance with the same
case insensitive name as a spec in the same lib.

Instances outside of a lib will have an identifier that is
project or system based.  It is illegal for a non-lib id
to contain "::" double colons; that format is reserved
for lib instances.

The scope of uniqueness for lib instances is global due to the
fact that lib names are [globally unique](Libs.md#names).  For
non-lib instances the id must at least be unique within the
containing dataset.

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
// separate slot tags with newline
@id: TypeSpec {
  slotA: valA
  slotB: valB
  ...
}

// or separate slot tags with comma
@id: TypeSpec {
  slotA: valA, slotB: valB, slot: valC
  ...
}

// the value can be omitted if it is a marker
@id: TypeSpec {
  markerA, markerB
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
using relative ids (absolute qnames are also permitted).  All
the non-maybe tags from the spec type are automatically added
into the instance.  The instances in the example above evaluate
to the following Haystack dicts:

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
from the instance specs `ph::Floor` and `ph::Room`.

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
Nested instances can have a slot name, a top-level id, or both.

Here is an example where the nested dicts have only a slot name:

```xeto
Toolbar: Dict
Button: Dict

@toolbar: Toolbar {
  save: Button { text:"Save" }
  exit: Button { text:"Exit" }
}
```

This maps to the following Haystack and JSON representations
where 'save' and 'exit' are just named slots that contain a
nested dict:

```trio
// Trio
id: @acme::toolbar
spec: @acme::Toolbar
exit: {spec:@acme::Button, text:"Exit" }
save: {spec:@acme::Button, text:"Save"}
```

```json
// JSON
"toolbar": {
    "id": "acme::toolbar"
    "spec": "acme::Toolbar",
    "save": {
      "spec": "acme::Button"
      "text": "Save",
    },
    "exit": {
      "spec": "acme::Button"
      "text": "Exit",
    },
  }
```

But we can also make those nested dicts first class instances with
an id using this syntax by adding an id after the slot name and
before the colon:

```xeto
// Xeto
@toolbar: Toolbar {
  save @save-button: Button { text:"Save" }
  exit @exit-button: Button { text:"Exit" }
}
```

```trio
// Haystack
id: @acme::toolbar
exit: {id:@acme::exit-button spec:@acme::Button text:"Exit"}
save: {id:@acme::save-button spec:@acme::Button text:"Save"}
spec: @acme::Toolbar
```

```json
// JSON
"toolbar": {
    "id": "acme::toolbar"
    "spec": "acme::Toolbar",
    "save": {
      "id": "acme::save-button",
      "spec": "acme::Button"
      "text": "Save",
    },
    "exit": {
      "id": "acme::exit-button",
      "spec": "acme::Button"
      "text": "Exit",
    },
  }
```

In many use cases if we don't care about the slot name, only
the nested id, in which case we can omit the slot name and
one is auto-generated for us:

```xeto
// Xeto
@toolbar: Toolbar {
  @save-button: Button { text:"Save" }
  @exit-button: Button { text:"Exit" }
}
```

```trio
// Haystack
id: @acme::toolbar
_0: {id:@acme::save-button spec:@acme::Button text:"Save"}
_1: {id:@acme::exit-button spec:@acme::Button text:"Exit"}
spec: @acme::Toolbar
````

```json
// JSON
"toolbar": {
    "id": "acme::toolbar"
    "spec": "acme::Toolbar",
    "_0": {
      "id": "acme::save-button",
      "spec": "acme::Button"
      "text": "Save",
    },
    "_1": {
      "id": "acme::exit-button",
      "spec": "acme::Button"
      "text": "Exit",
    },
  }
```

Note that in the reflection APIs the nested instances can be looked
up directly.  However, when exporting libs that contain nested instances
only top-level instances are included (with their nested instances).


