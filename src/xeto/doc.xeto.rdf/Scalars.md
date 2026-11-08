# Scalars

## Standard Scalar Slots

A plain Xeto scalar maps to an RDF typed literal. A typed literal contains a
lexical form and a datatype. The quoted text in Turtle is the lexical form; it
does not mean that every scalar is an `xsd:string`.

The scalar's effective Xeto type determines the datatype. The exporter does not
infer the datatype from the value's appearance. For example:

```xeto
Reading : Dict {
  label: Str
  count: Int
  value: Number
  ratio: Float
}

@reading1: Reading {
  label: "5"
  count: 5
  value: 5
  ratio: 5e-1
}
```

```turtle
ex:reading1 a sys:Entity, ex:Reading ;
  ex:Reading.label "5"^^xsd:string ;
  ex:Reading.count "5"^^xsd:integer ;
  ex:Reading.value "5.0"^^xsd:double ;
  ex:Reading.ratio "0.5"^^xsd:double .
```

The same type mapping selects the SHACL datatype:

| Xeto type | RDF datatype |
| --- | --- |
| `Str` | `xsd:string` |
| `Number` | `xsd:double` |
| `Float` | `xsd:double` |
| `Int` | `xsd:integer` |
| `Bool` | `xsd:boolean` |
| `Date` | `xsd:date` |
| `Time` | `xsd:time` |
| `DateTime` | `xsd:dateTime` |
| `Uri` | `xsd:anyURI` |
| `TimeZone` | `xsd:string` |

### SHACL Validation

A unit bearing `Number` is an exception, it maps to a `qudt:QuantityValue` node
rather than a plain literal, as described under
[Units and Quantities](UnitsAndQuantities.md#units-and-quantities).

`Float` is always unitless in Xeto and therefore never uses the QUDT quantity
mapping. Number and Float share the same finite and special double values:
`"NaN"^^xsd:double`, `"INF"^^xsd:double`, and `"-INF"^^xsd:double`. Negative
zero retains its sign. `NaN` cannot be used for `minVal` or `maxVal` because it
has no numeric order.

Numeric `minVal` and `maxVal` map to inclusive bounds.

```xeto
Sensor : Dict {
  sampleInterval: Int <minVal:1, maxVal:60>
}
```

```turtle
ex:Sensor a sh:NodeShape ;
  sh:targetClass ex:Sensor ;
  sh:property [
    sh:path ex:Sensor.sampleInterval ;
    sh:datatype xsd:integer ;
    sh:minInclusive 1 ;
    sh:maxInclusive 60 ;
    sh:minCount 1 ;
    sh:maxCount 1 ;
  ] .
```

String `minSize`, `maxSize`, and `nonEmpty` map to `sh:minLength`,
`sh:maxLength`, and `sh:pattern`. `nonEmpty` uses the pattern `\\S`, which
requires at least one non-whitespace character. Explicit `pattern` metadata
also maps to `sh:pattern`.

```xeto
Asset : Dict {
  dis: Str <nonEmpty, minSize:2, maxSize:40>
}
```

```turtle
ex:Asset a sh:NodeShape ;
  sh:targetClass ex:Asset ;
  sh:property [
    sh:path ex:Asset.dis ;
    sh:datatype xsd:string ;
    sh:minLength 2 ;
    sh:maxLength 40 ;
    sh:pattern "\\S" ;
    sh:minCount 1 ;
    sh:maxCount 1 ;
  ] .
```

An invariant slot value maps to `sh:hasValue` with the scalar's RDF datatype.
An ordinary default value does not constrain instance data.

```xeto
Thing : Dict {
  fixed: Str <invariant> "ok"
  label: Str "fallback"
}
```

```turtle
ex:Thing a sh:NodeShape ;
  sh:targetClass ex:Thing ;
  sh:property [
    sh:path ex:Thing.fixed ;
    sh:datatype xsd:string ;
    sh:hasValue "ok"^^xsd:string ;
    sh:minCount 1 ;
    sh:maxCount 1 ;
  ] ;
  sh:property [
    sh:path ex:Thing.label ;
    sh:datatype xsd:string ;
    sh:minCount 1 ;
    sh:maxCount 1 ;
  ] .
```

### Instance Data

Scalar values use the datatypes from the scalar mapping table. For example:

```xeto
Sensor : Dict {
  dis: Str
  priority: Int
  enabled: Bool
  commissioned: Date
}

@sensor1: Sensor {
  dis: "Zone Temp"
  priority: 5
  enabled: "true"
  commissioned: 2026-07-19
}
```

```turtle
ex:sensor1 a sys:Entity, ex:Sensor ;
  ex:Sensor.dis "Zone Temp"^^xsd:string ;
  ex:Sensor.priority "5"^^xsd:integer ;
  ex:Sensor.enabled "true"^^xsd:boolean ;
  ex:Sensor.commissioned "2026-07-19"^^xsd:date .
```

## Custom Scalar Datatypes

A custom Xeto scalar is a named type derived from `Scalar`, directly or through
another scalar type, with its own value constraints or encoding.

A concrete Xeto scalar without a specialized RDF mapping maps to a custom RDF
datatype. Its datatype IRI is the scalar spec's IRI in its versioned library
namespace. For example, `Code` in the example library maps to `ex:Code`.

Each value maps to an RDF literal whose datatype is that IRI and whose lexical
form is Xeto's string encoding of the value: `"AB"^^ex:Code`. This encoding
represents the scalar value; it does not imply that the value is a string.
The encoded form may differ from the original Xeto source spelling.

The mapping preserves type identity and encoded content. SHACL enforces
explicit constraints such as patterns, but does not reproduce native parsing,
comparison, execution, or resource lookup. These rules apply equally to
user-defined scalars and the built-in scalars listed below.

Types with an existing special mapping keep that mapping, including
refinements of [standard scalars](#standard-scalar-slots), enums, markers,
references, and units. The built-in scalars covered by the custom datatype
rule are listed [below](#built-in-and-library-scalars).

### Vocabulary

The vocabulary declares the scalar spec's IRI as an `rdfs:Datatype`:

```xeto
Code : Scalar <pattern:"[A-Z]{2}">

Asset : Dict {
  code: Code
}
```

```turtle
ex:Code a rdfs:Datatype ;
  rdfs:label "Code"@en .
```

The declaration names the datatype. The following shape enforces its pattern
when a value is used in `Asset.code`.

### SHACL Validation

A custom scalar slot checks the value's datatype with `sh:datatype` and its
declared pattern with `sh:pattern`. Required and optional slots follow the
ordinary cardinality rules.

```turtle
ex:Asset a sh:NodeShape ;
  sh:targetClass ex:Asset ;
  sh:property [
    sh:path ex:Asset.code ;
    sh:datatype ex:Code ;
    sh:pattern "[A-Z]{2}" ;
    sh:minCount 1 ;
    sh:maxCount 1 ;
  ] .
```

A custom scalar subtype maps to its own RDF datatype. Its shape preserves
both its declared patterns and those inherited from its base type:

```xeto
UpperCode : Scalar <pattern:"[A-Z]+">
FourCharacterCode : UpperCode <pattern:"[A-Z0-9]{4}">

Asset : Dict {
  code: FourCharacterCode
}
```

```turtle
ex:UpperCode a rdfs:Datatype ;
  rdfs:label "UpperCode"@en .

ex:FourCharacterCode a rdfs:Datatype ;
  rdfs:label "FourCharacterCode"@en ;
  rdfs:subClassOf ex:UpperCode .

ex:Asset a sh:NodeShape ;
  sh:targetClass ex:Asset ;
  sh:property [
    sh:path ex:Asset.code ;
    sh:datatype ex:FourCharacterCode ;
    sh:and (
      [ sh:pattern "[A-Z]+" ]
      [ sh:pattern "[A-Z0-9]{4}" ]
    ) ;
    sh:minCount 1 ;
    sh:maxCount 1 ;
  ] .
```

The slot's pattern constraints come from its declared scalar type, that type's
base types, and any pattern declared directly on the slot. Every applicable
pattern must be satisfied. Multiple distinct patterns map to separate shapes
within `sh:and`. SHACL permits only one `sh:pattern` value per shape under its
[multiple-parameter rule](https://www.w3.org/TR/shacl/#multiple-parameters).

An invariant custom scalar value maps to `sh:hasValue` with the slot's
datatype:

```xeto
Code : Scalar <pattern:"[A-Z]{2}">

FixedAsset : Dict {
  code: Code <invariant> "AB"
}
```

```turtle
ex:Code a rdfs:Datatype ;
  rdfs:label "Code"@en .

ex:FixedAsset a sh:NodeShape ;
  sh:targetClass ex:FixedAsset ;
  sh:property [
    sh:path ex:FixedAsset.code ;
    sh:datatype ex:Code ;
    sh:pattern "[A-Z]{2}" ;
    sh:hasValue "AB"^^ex:Code ;
    sh:minCount 1 ;
    sh:maxCount 1 ;
  ] .
```

`"AB"^^ex:Code` and `"AB"^^xsd:string` have the same text but different
datatypes. Only the first satisfies this invariant. An ordinary default does
not produce an `sh:hasValue` constraint.

### Instance Data

Using the `Code` and `Asset` declarations above:

```xeto
@asset1: Asset {
  code: "AB"
}
```

```turtle
ex:asset1 a sys:Entity, ex:Asset ;
  ex:Asset.code "AB"^^ex:Code .
```

For a slot declared with a custom scalar type, the slot's effective type
determines the literal's datatype. A slot declared as `UpperCode` therefore
uses `ex:UpperCode`, matching its `sh:datatype` constraint. The same rule
applies to supported list items with a declared scalar type and to scalar
slots in nested values.

An `Obj` slot does not declare a specific scalar type. The RDF datatype
therefore comes from the stored value's Xeto type. An explicitly typed `Code`
value retains that type; an ordinary string remains `xsd:string`:

```xeto
Envelope : Dict {
  value: Obj
}

@coded: Envelope {
  value: Code "AB"
}

@plain: Envelope {
  value: "AB"
}
```

```turtle
ex:coded a sys:Entity, ex:Envelope ;
  ex:Envelope.value "AB"^^ex:Code .

ex:plain a sys:Entity, ex:Envelope ;
  ex:Envelope.value "AB"^^xsd:string .
```

The export input must preserve a custom scalar value's type information.
If that information is missing, the exporter reports an error rather than
converting the custom value to an ordinary string.

### Built-in and Library Scalars

The same rule covers built-in and library scalars without another special
mapping. For example:

```turtle
sys:Version a rdfs:Datatype .
sys:Buf a rdfs:Datatype .

ex:record ex:version "1.2.3"^^sys:Version ;
  ex:payload "-_8"^^sys:Buf .
```

The text preserves what Xeto encoded. It does not make RDF tools understand
version ordering or binary data. `Duration` also uses this rule, even though
its Xeto base is `Number`.

| Other scalar | RDF representation | What RDF does not do |
| --- | --- | --- |
| `Version` | Xeto version text with datatype `sys:Version` | Compare version parts or check native parser limits |
| `Buf` | Xeto base64url text with datatype `sys:Buf` | Convert to `xsd:base64Binary` or decode the bytes |
| `Duration` | Xeto duration text with datatype `sys:Duration` | Perform time arithmetic or convert to a QUDT quantity |
| `Span` | Xeto span text with datatype `sys:Span` | Resolve relative dates or preserve context absent from the encoding |
| `Filter` | Xeto filter text with datatype `sys:Filter` | Parse, execute, or translate the filter to SPARQL |
| `LibDependVersions`, `LibFilePattern` | Selector text with the corresponding Xeto datatype | Resolve dependencies or inspect files |
| `None`, `NA` | The explicit value's Xeto encoding with its own datatype | Treat the value as omission, JSON null, or numeric NaN |
| `BuildVar` | A surviving placeholder's encoding with datatype `sys:BuildVar` | Perform build substitution |
| Other scalars, such as `pi::Icon`, `ph::Coord`, and user-defined `Code` | Xeto-encoded text with the scalar's versioned type IRI | Look up resources, split values into fields, or add type-specific parsing |

Values already replaced or removed by Xeto are not recreated during export.
A resolved build variable uses the mapping for its resulting value.

## Enums

An enum's entry names define its allowed string values unless an entry supplies
an explicit `key`.

```xeto
Suit : Enum {
  clubs
  diamonds
}

Card : Dict {
  suit: Suit
}
```

### Vocabulary

An enum spec is exported as a class, and the slot using it is exported as an
RDF property. Ordinary enum values are strings rather than RDF individuals,
so the vocabulary graph does not create a resource for each entry.

Some built-in enums have their own RDF mappings instead of this default string
mapping. For example, `Unit` and
[UnitQuantity](UnitsAndQuantities.md#standalone-unitquantity-values) represent their values as
resources, as described in their respective sections.

```turtle
ex:Suit a sys:Class, rdfs:Class ;
  rdfs:subClassOf sys:Enum ;
  rdfs:label "Suit"@en .

ex:Card.suit a rdf:Property ;
  rdfs:label "suit"@en .
```

### SHACL Validation

Ordinary enum slots are string-valued and use `sh:in` for the allowed values. For the
unkeyed `Suit` above, the entry names are used directly:

```turtle
ex:Card a sh:NodeShape ;
  sh:targetClass ex:Card ;
  sh:property [
    sh:path ex:Card.suit ;
    sh:datatype xsd:string ;
    sh:in ("clubs"^^xsd:string "diamonds"^^xsd:string) ;
    sh:minCount 1 ;
    sh:maxCount 1 ;
  ] .
```

If an enum entry declares `key` metadata, that key becomes the allowed value:

```xeto
Suit : Enum {
  clubs    <key:"Clubs">
  diamonds <key:"Diamonds">
}

Card : Dict {
  suit: Suit
}
```

The vocabulary declarations are unchanged. Only the `sh:in` values differ:

```turtle
ex:Card a sh:NodeShape ;
  sh:targetClass ex:Card ;
  sh:property [
    sh:path ex:Card.suit ;
    sh:datatype xsd:string ;
    sh:in ("Clubs"^^xsd:string "Diamonds"^^xsd:string) ;
    sh:minCount 1 ;
    sh:maxCount 1 ;
  ] .
```

The allowed values are therefore `"Clubs"` and `"Diamonds"`, not `"clubs"`
and `"diamonds"`.

Xeto enum definitions cannot be inherited. The Xeto compiler rejects a spec
that attempts to derive from an enum, so an enum slot's allowed values come
from the enum named by that slot. There is no additional enum-inheritance rule
in the RDF mapping.

### Instance Data

For the unkeyed enum, an instance uses the entry name as its string value:

```xeto
@card1: Card {
  suit: "clubs"
}
```

```turtle
ex:card1 a sys:Entity, ex:Card ;
  ex:Card.suit "clubs"^^xsd:string .
```

For the keyed definition, the instance and RDF literal use the explicit key:

```xeto
@card2: Card {
  suit: "Clubs"
}
```

```turtle
ex:card2 a sys:Entity, ex:Card ;
  ex:Card.suit "Clubs"^^xsd:string .
```

