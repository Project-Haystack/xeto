# Overview

**Jeto** is xeto-typed JSON: the spec supplies the types, so the values don't
have to declare them.  It is the representation of Xeto
[instances](Instances.md) in the widely used data interchange format
[JSON](https://en.wikipedia.org/wiki/JSON).

Jeto reads as ordinary JSON.  A date is the string `"2024-11-26"` and a dict is
a plain object, so a consumer which does not know Xeto still sees data it can
work with.

Jeto is represented by the file spec [sys.files::JetoFile].

Jeto is distinct from [Hayson](ph.doc::Hayson), the Haystack 4 JSON encoding,
which is self-typed JSON: each value declares its own kind.  The two are
different formats with different goals.  Hayson makes every value
self-describing at the cost of verbosity, and needs no schema to read back.
Hayson only supports the original fixed scalar types, while Jeto supports the full
Xeto type system. Jeto stays terse by leaning on the spec, and boxes only the
values a spec does not cover.

## Scalars

JSON supports a very limited type system, which requires significant type
erasure.  Jeto opts for a clean, simple mapping where most scalars are mapped
to a string (versus mapping scalars to JSON objects).  The following mapping is
used:

| Xeto        | JSON                                        |
| ----        | ----                                        |
| sys::Bool   | boolean                                     |
| sys::Int    | number                                      |
| sys::Float  | number                                      |
| sys::Number | number (if no unit)                         |
| sys::Number | string (if there is a unit or NaN/-INF/INF) |
| sys::Buf    | string (base64url, unpadded)                |
| sys::Scalar | string (all other scalars encode as string) |

The singleton scalars encode as their own string forms: Marker as `"✓"`, None
as `"∅"` and NA as `"NA"`.  A JSON null never means NA.

## Type Resolution

A JSON string carries no type information, so a decoder must determine the
intended Xeto type from context.  Types are never inferred from the content of
a string.  In a position with no type, `"2024-11-26"` decodes as a Str, not a
Date, and `"✓"` decodes as a Str, not a Marker.  Tag names carry no meaning
either: `id` decodes as a Ref only where the spec declares one.

Resolution has two halves: the type a position expects, and what the encoded
value does with it.

A position's expected type is the first of these which applies:

1. For a grid cell, the column's `of`
2. The type declared for this member by the containing dict's spec.  That spec
   comes from the dict's own `spec` property, from a grid's default row spec,
   or from the type the enclosing spec declares for the dict itself
3. None, which leaves the position untyped

Three JSON forms carry their own type and keep it whatever the position
expects:

| JSON         | Decodes as                                        |
| ----         | ----                                              |
| boxed scalar | the type named by its `spec`                      |
| true / false | Bool                                              |
| number       | Int with no fraction or exponent, otherwise Float |

Such a disagreement is not an error.  A boxed Date in a slot declared Number
decodes as a Date.  A number in a Bool column decodes as an Int.

Numbers have one exception.  Int, Float and Number are interchangeable, so
where the expected type is one of the three the number takes it.  Against any
other type it keeps its lexical form.

A JSON string has no type of its own.  It takes the expected type, and decodes
as a Str where there is none.

A unitless Number does not survive a position with no type.  It encodes as a
bare JSON number and decodes as an Int or Float.  It must be boxed, or its
container must declare a type.

## Boxed Scalars

Any scalar may instead be encoded as a JSON object which names its own type:

```json
{"val":"72°F", "spec":"sys::Number"}
```

The `val` property is always a JSON string, even for types which have a native
JSON form.  A boxed scalar requires no context and is legal anywhere a scalar
may appear: slot value, list element, grid cell or meta tag.

An object whose `spec` resolves to a non-scalar type is an ordinary dict, not
a boxed scalar.

A boxed Ref may also carry its display string:

```json
{"val":"xyz-123", "spec":"sys::Ref", "dis":"Carytown"}
```

Encoders support three boxing modes:

| Mode | Behavior                                            |
| ---- | ----                                                |
| none | never box                                           |
| auto | box only where the plain form would not decode back |
| all  | box every scalar                                    |

The default is auto.  The none mode is lossy in any position with no type:
markers, refs, dates and unitless numbers all decode as Str.  The auto mode
is lossless.

The reserved structural properties are never boxed in any mode: a dict's
`spec`, a column's `name` and `of`, and a grid's `spec` and `of`.

## Dicts

Dicts are mapped as JSON objects.  A dict with a known type includes a 'spec'
property with the type qname.  For example, for a Dict conforming to the Xeto
spec `foo::Bar`:

```json
{
  "a":123,
  "b":"xyz",
  "spec":"foo::Bar"
}
```

Dicts with no known type omit the 'spec' property.  The 'spec' property is
reserved: it always holds a type qname and is never one of the dict's own
tags.

A JSON null means the tag is absent.  The object `{"a":null}` and an object
with no `a` entry decode to the same dict.

## Lists

Lists are mapped as JSON arrays:

```json
["a", "b", "c"]
```

Null elements are retained.

## Grids

Each Jeto grid consists of a JSON object with the following entries:

- A `spec` entry which identifies them as a Grid, or subclass thereof.
- A `meta` entry which contains the metadata for the grid itself.
- A `cols` entry which contains the name, type and metadata for each column.
- A `rows` entry which contains each row in Dict format.

For example:

```json
{
  "spec":"sys::Grid",
  "meta":{
    "foo":"quux"
  },
  "cols":[
    {
      "name":"a"
    },
    {
      "name":"b",
      "meta":{
        "dis":"B"
      }
    }
  ],
  "rows":[
    {
      "a":0,
      "b":"x"
    },
    {
      "a":1,
      "b":"y"
    }
  ]
}
```

Grids may declare types for their cells in three places:

- A column may declare an `of` entry beside its `name`, which gives a type to
  that column's cells.
- A grid may declare an `of` entry beside its `spec`, which gives a default
  spec to every row.
- A row may include a `spec` property, which overrides the default row spec
  for that row.  The rows of a grid need not all be the same type.

Both `of` entries sit beside the thing they describe rather than inside its
`meta`, because they are structural rather than domain tags.  `meta` holds
only domain tags, at both the grid and the column level.

For example:

```json
{
  "spec":"sys::Grid",
  "of":"ph::Equip",
  "cols":[
    {
      "name":"ts",
      "of":"sys::DateTime"
    },
    {
      "name":"v0",
      "of":"sys::Number"
    }
  ],
  "rows":[
    {
      "ts":"2026-08-03T09:00:00-04:00 New_York",
      "v0":"72.5"
    },
    {
      "spec":"ph::Vav",
      "ts":"2026-08-03T10:00:00-04:00 New_York",
      "v0":73
    }
  ]
}
```

A grid whose `spec` names a Grid subclass takes its default row spec from that
subclass.  The grid `of` entry is for grids whose row spec is not named by a
subclass.

Encoders carry these entries but do not synthesize them.  A producer which
relies on column typing declares `of` itself; otherwise the cells which need
it are boxed.

## Specs

Xeto specs are represented in JSON via the [JSON
Schema](https://json-schema.org/) specification, using the
[2020-12](https://json-schema.org/draft/2020-12/schema) dialect.  This allows
for Jeto instances to be validated against Xeto specs without
resorting to Xeto-specific tools.

A generated schema types each scalar as a string, so it describes unboxed
instances.  A boxed scalar is a JSON object where the schema expects a string
and will not validate.  Instances intended for schema validation are encoded
with boxing off.

Consider the following Xeto specs:

```xeto
Order : Entity {
  order
  customerName: Str
  orderDate: DateTime
  orderType: OrderType
  items: List <of:Product>
}

OrderType: Enum {
  kitchen
  bathRoom
  livingRoom
  secretLab
}

Product : Dict {
  product
  name: Str
  price: Float
}
```

The JSON Schema representation of these specs is as follows.

- All of the definitions are stored in the `$defs` section of the schema, keyed
by flat qname such as `sys.Ref`.  There is no version stamp: a single document
is one coherent namespace and cannot hold two versions of a library.
- `Str`, `Bool`, and `Int` are stamped inline where they are used.  Every other
type gets a `$defs` entry and is referenced with `$ref`.
- Scalars are represented by a `string` type with an anchored `pattern` regex.
Some also carry a `format` annotation alongside the pattern, and single-valued
scalars such as `Marker` use `const` instead.
- `Number` and `Float` are unions, because their wire form is not one JSON
type: a bare JSON number, the quoted specials `"NaN"`, `"INF"`, and `"-INF"`,
and for `Number` a quoted string when the value carries a unit.
- Enums (a special case of Scalars) are represented by a `string` type with an
`enum` array.
- Lists are represented by an `array` type with a reference to the type of the
array's elements.
- Dicts are represented by an `object` type which defines the dict's slots
as a set of `properties` entries.  If the spec is a subclass of Dict, then
the `allOf` keyword is used to provide a reference to the parent class.
- A maybe slot is represented by omitting it from `required`.  Null is not
used: in a Dict an absent tag and a null tag are the same thing.
- Xeto type information which JSON Schema cannot express is attached as an
`x-xeto` annotation carrying the native `::` qname and the spec's own meta.
Unknown keywords are annotations rather than errors in 2020-12, so a generic
validator ignores it while Xeto-aware tooling can recover the real type.

Here is an example of what the example specs look like as a JSON-Schema:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "hx.test.xeto-4.0.6",
  "title": "Haxall test library",
  "$defs": {
    "sys.Marker": {
      "const": "\u2713",
      "description": "Marker labels a dict with typing information",
      "x-xeto": {
        "spec": "sys::Marker"
      }
    },
    "sys.Ref": {
      "type": "string",
      "pattern": "^(?:[a-zA-Z\\d\\._~]*)$",
      "description": "Reference to another dict.  The `of` meta defines target type",
      "x-xeto": {
        "spec": "sys::Ref"
      }
    },
    "sys.Entity": {
      "description": "Top-level dicts with a unique identifier.",
      "x-xeto": {
        "spec": "sys::Entity",
        "abstract": true
      },
      "additionalProperties": true,
      "type": "object",
      "properties": {
        "id": {
          "description": "Unique identifier for entity in its project",
          "$ref": "#/$defs/sys.Ref"
        },
        "spec": {
          "description": "Type of this entity",
          "x-xeto": {
            "of": "sys::Spec"
          },
          "$ref": "#/$defs/sys.Ref"
        }
      },
      "required": [
        "id"
      ]
    },
    "sys.Float": {
      "anyOf": [
        {
          "type": "number"
        },
        {
          "enum": [
            "NaN",
            "INF",
            "-INF"
          ]
        }
      ],
      "description": "Unitless floating point number",
      "x-xeto": {
        "spec": "sys::Float",
        "unitless": true
      }
    },
    "sys.DateTime": {
      "type": "string",
      "pattern": "^(?:\\d{4}-\\d{2}-\\d{2}T\\d{2}:\\d{2}:\\d{2}(\\.\\d+)*[Z+-][\\d:]*[ ]*[-+a-zA-Z_\\d]*)$",
      "description": "ISO 8601 timestamp followed by timezone identifier",
      "x-xeto": {
        "spec": "sys::DateTime"
      }
    },
    "hx.test.xeto.OrderType": {
      "x-xeto": {
        "spec": "hx.test.xeto::OrderType"
      },
      "type": "string",
      "enum": [
        "kitchen",
        "bathRoom",
        "livingRoom",
        "secretLab"
      ]
    },
    "hx.test.xeto.Product": {
      "x-xeto": {
        "spec": "hx.test.xeto::Product"
      },
      "additionalProperties": true,
      "type": "object",
      "properties": {
        "product": {
          "$ref": "#/$defs/sys.Marker"
        },
        "price": {
          "$ref": "#/$defs/sys.Float"
        },
        "name": {
          "type": "string"
        }
      },
      "required": [
        "product",
        "name",
        "price"
      ]
    },
    "hx.test.xeto.Order": {
      "allOf": [
        {
          "$ref": "#/$defs/sys.Entity"
        },
        {
          "additionalProperties": true,
          "type": "object",
          "properties": {
            "orderType": {
              "$ref": "#/$defs/hx.test.xeto.OrderType"
            },
            "id": {
              "description": "Unique identifier for entity in its project",
              "$ref": "#/$defs/sys.Ref"
            },
            "orderDate": {
              "$ref": "#/$defs/sys.DateTime"
            },
            "items": {
              "type": "array",
              "items": {
                "$ref": "#/$defs/hx.test.xeto.Product"
              }
            },
            "spec": {
              "description": "Type of this entity",
              "x-xeto": {
                "of": "sys::Spec"
              },
              "$ref": "#/$defs/sys.Ref"
            },
            "customerName": {
              "type": "string"
            },
            "order": {
              "$ref": "#/$defs/sys.Marker"
            }
          },
          "required": [
            "id",
            "order",
            "customerName",
            "orderDate",
            "orderType",
            "items"
          ]
        }
      ],
      "x-xeto": {
        "spec": "hx.test.xeto::Order"
      }
    }
  }
}
```

