# Overview

Xeto supports the representation of [instances](Instances.md) and
[specs](Specs.md) via the widely used data interchange format
[JSON](https://en.wikipedia.org/wiki/JSON).

## Scalars

JSON supports a very limited type system, which requires significant type
erasure.  We opt for a clean, simple JSON mapping where most scalars are mapped
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

The type of a value comes from the first of these which applies:

1. A boxed scalar, which names its own type
2. A grid column's `of`
3. The spec of the containing dict, which may come from its `spec` property,
   from a grid's default row spec, or from the enclosing member's declared type
4. Otherwise a JSON string decodes as a Str

A boxed scalar or a native JSON number or boolean takes priority over the
surrounding context.  Such a disagreement is not an error.  A boxed Date in a
slot declared Number decodes as a Date.

A JSON number decodes by its lexical form: Int when it has no fraction or
exponent, Float otherwise.  Int, Float and Number are interchangeable, so a
number takes whichever of the three the position declares.  Against any other
type the number keeps its lexical form.  A number in a Bool column decodes as
an Int.

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

The default is none.  It is lossy in any position with no type: markers, refs,
dates and unitless numbers all decode as Str.  The auto mode is lossless.

The reserved structural properties are never boxed in any mode: a dict's
`spec`, a column's `name` and `of`, and a grid's `spec`.

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

Each JSON-formatted grid consists of a JSON object with the following
entries:

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
  that column's cells.  It sits at the column level rather than inside `meta`
  because it is structural rather than a domain tag.
- The grid `meta` may declare an `of` entry, which gives a default spec to
  every row.
- A row may include a `spec` property, which overrides the default row spec
  for that row.  The rows of a grid need not all be the same type.

For example:

```json
{
  "spec":"sys::Grid",
  "meta":{
    "of":"ph::Equip"
  },
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
subclass.  The `meta` `of` entry is for grids whose row spec is not named by a
subclass.

Encoders carry these entries but do not synthesize them.  A producer which
relies on column typing declares `of` itself; otherwise the cells which need
it are boxed.

## Specs

Xeto specs are represented in JSON via the [JSON
Schema](https://json-schema.org/) specification.  This allows for
JSON-formatted Xeto instances to be validated against Xeto specs without
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

- All of the definitions are stored in the `$defs` section of the schema.
- The `$defs` section of the schema has sub-entries per xeto library.
- Scalars are represented by a `string` type with a `pattern` regex pattern, unless they correspond to the JSON types `boolean`, `integer`, or `number`.
- Enums (a special case of Scalars) are represented by a `string` type with an
`enum` array.
- List are represented by an `array` type with a reference to the type of the
array's elements.
- Dicts are represented by an `object` type with defines the dict's slots
as a set of `properties` entries.  If the spec is a subclass of Dict, then
the `allOf` keyword is used to provided a reference to the parent class.

Here is an example of what the example specs look like as a JSON-Schema:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "hx.test.xeto.schema-4.0.4",
  "title": "Haxall test library",
  "$defs": {
    "sys-5.0.0": {
      "Entity": {
        "additionalProperties": true,
        "type": "object",
        "properties": {
          "id": {
            "$ref": "#/$defs/sys-5.0.0/Ref"
          },
          "spec": {
            "$ref": "#/$defs/sys-5.0.0/Ref"
          }
        },
        "required": [
          "id"
        ]
      },
      "Ref": {
        "pattern": "[a-zA-Z\\d\\._~]*",
        "type": "string"
      },
      "Marker": {
        "pattern": "\u2713",
        "type": "string"
      },
      "DateTime": {
        "pattern": "\\d{4}-\\d{2}-\\d{2}T\\d{2}:\\d{2}:\\d{2}(\\.\\d+)*[Z+-][\\d:]*[ ]*[-+a-zA-Z_\\d]*",
        "type": "string"
      }
    },
    "hx.test.xeto.schema-4.0.4": {
      "Order": {
        "allOf": [
          {
            "$ref": "#/$defs/sys-5.0.0/Entity"
          },
          {
            "additionalProperties": true,
            "type": "object",
            "properties": {
              "orderType": {
                "$ref": "#/$defs/hx.test.xeto.schema-4.0.4/OrderType"
              },
              "orderDate": {
                "$ref": "#/$defs/sys-5.0.0/DateTime"
              },
              "items": {
                "type": "array",
                "items": {
                  "$ref": "#/$defs/hx.test.xeto.schema-4.0.4/Product"
                }
              },
              "customerName": {
                "type": "string"
              },
              "order": {
                "$ref": "#/$defs/sys-5.0.0/Marker"
              }
            },
            "required": [
              "order",
              "customerName",
              "orderDate",
              "orderType",
              "items"
            ]
          }
        ]
      },
      "OrderType": {
        "type": "string",
        "enum": [
          "kitchen",
          "bathRoom",
          "livingRoom",
          "secretLab"
        ]
      },
      "Product": {
        "additionalProperties": true,
        "type": "object",
        "properties": {
          "product": {
            "$ref": "#/$defs/sys-5.0.0/Marker"
          },
          "price": {
            "type": "number"
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
      }
    }
  }
}
```

