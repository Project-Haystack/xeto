# Overview

Xeto supports the representation of [instances](Instances.md) and
[specs](Specs.md) via the widely used data interchange format 
[JSON](https://en.wikipedia.org/wiki/JSON).

## Scalars

JSON supports a very limited type system, which requires significant type
erasure.  We opt for a clean, simple JSON mapping where most scalars are mapped
to a string (versus mapping scalars to JSON objects).  The following mapping is
used:

```
| Xeto        | JSON                                        |
| ----        | ----                                        |
| sys::Bool   | boolean                                     |
| sys::Int    | number                                      |
| sys::Float  | number                                      |
| sys::Number | number (if no unit)                         |
| sys::Number | string (if there is a unit or NaN/-INF/INF) |
| sys::Scalar | string (all other scalars encode as string) |
```

## Dicts

Dicts are mapped as JSON objects. All dicts include a 'spec' property
with the type qname. For example, for a Dict conforming to the Xeto spec
`foo::Bar`:

```json
  {
    "a":123,
    "b":"xyz",
    "spec":"foo::Bar"
  }
```

## Lists

Lists are mapped as JSON arrays: 

```json
  ["a", "b", "c"]
```

## Grids

Each JSON-formatted grid consists of a JSON object with the following 
entries:

- A `spec` entry which identifies them as a Grid, or subclass thereof.  
- A `meta` entry which contains the metadata for the grid itself.
- A `cols` entry which contains the name and metadata for each column.
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


## Specs

Xeto specs are represented in JSON via the [JSON
Schema](https://json-schema.org/) specification.  This allows for
JSON-formatted Xeto instances to be validated against Xeto specs without
resorting to Xeto-specific tools.

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
