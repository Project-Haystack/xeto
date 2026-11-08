# Units and Quantity Values

## Units and Quantities

### Unit and Quantity Correspondence

Xeto defines `sys::Unit` from the standard Project Haystack unit database. Each
unit has a full name and may have one or more symbol aliases. For example,
`square_meter` has the symbol `m²`, while `hour` has both `hr` and `h`. When a
unit has multiple symbols, the last symbol in the standard database is its
preferred display form.

The complete Xeto unit catalog is defined by the
[Xeto `sys::Unit` source](https://github.com/Project-Haystack/xeto/blob/master/src/xeto/sys/units.xeto).

This specification classifies every unit and quantity in the current Xeto
catalog through machine-readable properties files:

- [Xeto unit to QUDT unit or currency](https://github.com/Project-Haystack/xeto/blob/master/src/xeto/sys.rdf/qudt-units.props)
- [Xeto quantity to QUDT quantity kind](https://github.com/Project-Haystack/xeto/blob/master/src/xeto/sys.rdf/qudt-quantities.props)
- [Xeto units without a QUDT mapping](https://github.com/Project-Haystack/xeto/blob/master/src/xeto/sys.rdf/qudt-unmapped-units.props)
- [Xeto quantities without a QUDT mapping](https://github.com/Project-Haystack/xeto/blob/master/src/xeto/sys.rdf/qudt-unmapped-quantities.props)

The QUDT release, source graphs, revision, and namespace IRIs used by these
tables are pinned in the
[Xeto-to-QUDT mapping metadata](https://github.com/Project-Haystack/xeto/blob/master/src/xeto/sys.rdf/qudt-meta.props).
Changing that file is an explicit mapping-version upgrade.

Mapped rows have the form `xetoName=qudtLocalName`. A quantity may list more
than one comma-separated QUDT quantity kind when QUDT uses distinct kinds for
units that Xeto groups together. Unmapped rows use
`xetoName=reasonCode|reviewNote`. The fixed QUDT namespaces are
`http://qudt.org/vocab/unit/` for physical units,
`http://qudt.org/vocab/currency/` for currencies, and
`http://qudt.org/vocab/quantitykind/` for quantity kinds. For example:

```properties
fahrenheit=DEG_F
us_dollar=USD
temperature=Temperature
velocity=Velocity,Speed
belarussian_ruble=noEquivalent|QUDT 2.1 currency graph does not contain obsolete ISO 4217 code BYR
```

The Xeto unit's quantity determines the target namespace. Thus
`fahrenheit=DEG_F` maps to `unit:DEG_F`, while the Xeto currency unit
`us_dollar=USD` maps to `currency:USD`. QUDT types currency resources as
`qudt:CurrencyUnit`, a subclass of `qudt:Unit`.

Xeto units can be written using their full names or any symbol defined for the
same unit. The unit mapping properties file contains one entry per unit, keyed
by its full Xeto name. Before applying the mapping, an implementation resolves
any symbol to the full Xeto unit name, then maps it to the corresponding QUDT
unit. A Xeto `quantity` constraint is looked up by its `sys::UnitQuantity` name
in the quantity-kind mapping.

The mapping preserves the original unit and numeric value. It does not convert
a number into another unit. For example, `32°F` maps to the value `32` with
unit `unit:DEG_F`; it is not converted to `0°C`.

Every current Xeto unit and quantity appears in exactly one mapped or unmapped
inventory. An exporter does not guess a QUDT resource from a similar name or
symbol. A value classified as unmapped is an export error; its inventory row
records why no correspondence is defined.

QUDT publishes its unit definitions in the
[QUDT Units Vocabulary](https://www.qudt.org/doc/DOC_VOCAB-UNITS.html) and
provides downloadable vocabulary graphs through the
[QUDT catalog](https://www.qudt.org/catalog/qudt-catalog.html).

### SHACL Validation

A `Unit` slot contains a canonical QUDT unit IRI. The shape requires
`qudt:Unit`. If the slot specifies a quantity, the unit's QUDT quantity kind
must match it directly or through the hierarchy described below. The validator
reads those relationships from the separately loaded QUDT vocabulary graph.

```xeto
Sensor : Dict {
  unit: Unit <quantity:"temperature">
}
```

```turtle
ex:Sensor a sh:NodeShape ;
  sh:targetClass ex:Sensor ;
  sh:property [
    sh:path ex:Sensor.unit ;
    sh:class qudt:Unit ;
    sh:node [
      sh:property [
        sh:path (
          qudt:hasQuantityKind
          [ sh:zeroOrMorePath skos:broader ]
        ) ;
        sh:hasValue quantitykind:Temperature ;
      ]
    ] ;
    sh:minCount 1 ;
    sh:maxCount 1 ;
  ] .
```

#### Quantity Kind Hierarchy

The standard Xeto unit catalog groups each unit under a quantity. For example,
`gallon`, `fluid_ounce`, and `imperial_gallon` are in Xeto's `volume` group.
The quantity-kind mapping relates that Xeto quantity to
`quantitykind:Volume`:

```properties
volume=Volume
```

QUDT gives some units a more specific quantity kind. In the pinned QUDT graph,
the US gallon is classified as `LiquidVolume`, and QUDT identifies
`LiquidVolume` as a narrower concept beneath `Volume`:

```turtle
unit:GAL_US qudt:hasQuantityKind quantitykind:LiquidVolume .

quantitykind:LiquidVolume
  skos:broader quantitykind:Volume .
```

For this mapping, a QUDT quantity kind satisfies a Xeto quantity constraint
when it is the mapped kind or when following one or more QUDT `skos:broader`
relationships reaches the mapped kind. Therefore `unit:GAL_US` satisfies
Xeto's `volume` constraint. Checking only the unit's direct QUDT quantity kind
would incorrectly reject it.

This is a rule of this specification, not a requirement imposed on every QUDT
consumer. The SHACL `sh:zeroOrMorePath` performs the hierarchy traversal
directly, so validation does not require a SKOS reasoner. The separately loaded
QUDT graph must provide both the units' `qudt:hasQuantityKind` statements and
the quantity kinds' `skos:broader` statements. `skos:broader` is defined by the
[W3C SKOS recommendation](https://www.w3.org/TR/skos-reference/#semantic-relations).

Only Xeto units listed in the
[unit mapping](https://github.com/Project-Haystack/xeto/blob/master/src/xeto/sys.rdf/qudt-units.props)
can be exported. Encountering a unit in the unsupported inventory is an export
error.

A number constrained by `unit` or `quantity` is represented as a
`qudt:QuantityValue` node rather than as a plain numeric literal. Numeric range
constraints apply to `qudt:numericValue`; unit constraints apply to
`qudt:unit`.

```xeto
Meter : Dict {
  percent: Number <minVal:0, maxVal:100, unit:"%">
  power: Number <quantity:"power">
}
```

The `percent` shape is:

```turtle
ex:Meter a sh:NodeShape ;
  sh:targetClass ex:Meter ;
  sh:property [
    sh:path ex:Meter.percent ;
    sh:minCount 1 ;
    sh:maxCount 1 ;
    sh:node [
      sh:class qudt:QuantityValue ;
      sh:property [
        sh:path qudt:numericValue ;
        sh:datatype xsd:double ;
        sh:minInclusive "0.0"^^xsd:double ;
        sh:maxInclusive "100.0"^^xsd:double ;
        sh:minCount 1 ;
        sh:maxCount 1 ;
      ] ;
      sh:property [
        sh:path qudt:unit ;
        sh:class qudt:Unit ;
        sh:hasValue unit:PERCENT ;
        sh:minCount 1 ;
        sh:maxCount 1 ;
      ]
    ] ;
  ] .
```

The `power` shape has the same structure but replaces the exact unit with a
`qudt:hasQuantityKind quantitykind:Power` constraint.

### Instance Data

A `Unit` value becomes its canonical QUDT unit IRI:

```turtle
ex:sensor1 ex:Sensor.unit unit:DEG_F .
```

A unit-bearing number becomes a `qudt:QuantityValue` node:

```xeto
@meter1: Meter {
  percent: 50%
  power: 1kW
}
```

```turtle
ex:meter1 a sys:Entity, ex:Meter ;
  ex:Meter.percent [
    a qudt:QuantityValue ;
    qudt:numericValue "50.0"^^xsd:double ;
    qudt:unit unit:PERCENT
  ] ;
  ex:Meter.power [
    a qudt:QuantityValue ;
    qudt:numericValue "1.0"^^xsd:double ;
    qudt:unit unit:KiloW
  ] .
```

Currency units follow the same Number rule:

```xeto
Invoice : Dict {
  price: Number <unit:"\$">
}

@invoice1: Invoice {
  price: 19.99$
}
```

```turtle
ex:invoice1 ex:Invoice.price [
  a qudt:QuantityValue ;
  qudt:numericValue "19.99"^^xsd:double ;
  qudt:unit currency:USD
] .
```

The exporter does not round currency to a fixed number of decimal places or
claim exact decimal arithmetic that the Xeto value does not provide. An
application that requires accounting semantics must define its own rounding
or integer-minor-unit convention before RDF export.

The QUDT declarations and quantity-kind facts referenced by these values are
loaded separately during validation as described under
[Metadata and External Vocabularies](Metadata.md#metadata-and-external-vocabularies).

## Standalone UnitQuantity Values

A standalone `UnitQuantity` value maps to `sys:UnitQuantity.<key>`, using the
exact, case-sensitive enum key and the versioned `sys` namespace. One Xeto
quantity can correspond to several QUDT quantity kinds, or have no QUDT match.
Giving it its own resource preserves one Xeto value without choosing a single
QUDT match or turning it into several values.

```xeto
Reading : Dict {
  quantity: UnitQuantity
}

@reading1: Reading { quantity: "energy" }
```

```turtle
ex:reading1 a sys:Entity, ex:Reading ;
  ex:Reading.quantity sys:UnitQuantity.energy .

sys:UnitQuantity.energy a sys:UnitQuantity ;
  rdfs:label "energy"@en ;
  rdfs:seeAlso quantitykind:Energy, quantitykind:MomentOfForce,
    quantitykind:Torque .
```

Each quantity resource has a label and `rdfs:seeAlso` links to its matching
QUDT quantity kinds, of which there may be several. For example, Xeto's
`energy` category maps to both `quantitykind:Energy` and `quantitykind:Torque`.
These links do not assert equivalence: the link to `quantitykind:Torque` does
not mean that every instance with quantity `energy` measures torque. A quantity without
a QUDT match, such as `powerByVolumetricFlow`, has its own resource and label,
but no QUDT links.

All standalone `UnitQuantity` slots reference a shared
`sys:UnitQuantityShape`:

```turtle
sys:UnitQuantityShape a sh:NodeShape ;
  sh:nodeKind sh:IRI ;
  sh:in (sys:UnitQuantity.energy sys:UnitQuantity.temperature # ...
  ) .
```

Each exported schema graph containing a `UnitQuantity` slot includes this
shape once. It requires the value to be an IRI from the quantity catalog.
The `sh:in` list contains a resource for every effective `sys::UnitQuantity`
enum entry, using the versioned `sys` namespace. The example above abbreviates
that list.

The property shape for each standalone `UnitQuantity` slot references the
shared shape and defines that slot's cardinality and invariant constraints.
For example, `Reading.quantity`
requires exactly one quantity value:

```turtle
ex:Reading a sh:NodeShape ;
  sh:targetClass ex:Reading ;
  sh:property [
    sh:path ex:Reading.quantity ;
    sh:node sys:UnitQuantityShape ;
    sh:minCount 1 ;
    sh:maxCount 1
  ] .
```

An optional slot omits `sh:minCount`. An invariant adds, for example,
`sh:hasValue sys:UnitQuantity.energy`.

