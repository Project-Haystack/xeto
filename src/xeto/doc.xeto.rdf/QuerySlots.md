# Query Slot Mapping

A Xeto `Query` is computed navigation, not a stored required field. It selects
records by following references from a starting record. This chapter calls a
SPARQL query that performs that selection over exported RDF data a
**SPARQL selection query**. The term Query refers to the Xeto definition;
SPARQL selection query refers to its SPARQL translation.

Query traversal uses the [reference-field metadata](References.md#reference-field-metadata)
in exported instance data to distinguish Refs from other IRI-valued fields.

## SHACL Validation

Query slots without named body requirements impose no validation constraints. Their
`via` or `inverse` metadata is retained as a SHACL property path, without
`sh:class`, `sh:minCount`, or `sh:maxCount`. The `of` type selects query results;
it does not require every record along the path to have that type.

### Supported Query Paths

```xeto
Equip : Dict {
  plainPoints: Query<of:Point>
  points: Query<of:Point, inverse:"Point.equips">
}

Point : Dict {
  equipRef: Ref<of:Equip>
  equips: Query<of:Equip, via:"equipRef+">
}

Node : Dict {
  next: Ref<of:Node, maybe>
  reachable: Query<of:Node, via:"next*">
}
```

```turtle
ex:Equip a sh:NodeShape ;
  sh:targetClass ex:Equip ;
  sh:property [
    # Equip.plainPoints
    sh:path ex:Equip.plainPoints ;
  ] ;
  sh:property [
    # Equip.points: inverse of Point.equips
    sh:path [
      sh:oneOrMorePath [ sh:inversePath ex:Point.equipRef ]
    ] ;
  ] .

ex:Point a sh:NodeShape ;
  sh:targetClass ex:Point ;
  sh:property [
    sh:path ex:Point.equipRef ;
    sh:class ex:Equip ;
    sh:nodeKind sh:IRI ;
    sh:minCount 1 ;
    sh:maxCount 1 ;
  ] ;
  sh:property [
    # Point.equips: follows equipRef one or more times
    sh:path [ sh:oneOrMorePath ex:Point.equipRef ] ;
  ] .

ex:Node a sh:NodeShape ;
  sh:targetClass ex:Node ;
  sh:property [
    sh:path ex:Node.next ;
    sh:class ex:Node ;
    sh:nodeKind sh:IRI ;
    sh:maxCount 1 ;
  ] ;
  sh:property [
    # Node.reachable: follows next zero or more times
    sh:path [ sh:zeroOrMorePath ex:Node.next ] ;
  ] .
```

A Query without named body requirements does not require a particular number
of matches. Its path may reach no records, one record, or many records. Only
records that fit the Query's `of` type are returned; reaching other types does
not invalidate the record containing the Query. A `+` path follows one or more
references and may pass through records that do not fit `of`. Cycles are allowed,
and each matching record is returned once.

The Query's SHACL property path imposes no class or cardinality constraint.
Constraints on the stored Ref or MultiRef remain unchanged: `Ref<of:Point>`
still requires its referenced record to have class Point. Query execution and
validation of the returned records are separate operations.

Given only this instance relationship:

```turtle
ex:point1 ex:Point.equipRef ex:equip1 .
```

the inverse path can navigate from `ex:equip1` to `ex:point1`. The instance
graph does not need this materialized reverse triple:

```turtle
ex:equip1 ex:Equip.points ex:point1 .
```

A plain query without `via` or `inverse` uses its slot property as `sh:path`,
but still has no cardinality constraint. Exporting schema SHACL and compiling
Query-body validation are separate operations. The schema-only exporter does
not yet support Query bodies; the Query compiler produces the SPARQL and SHACL
needed to check required and optional body members. See
[Query-Body Validation](Queries.md#query-body-validation) and
[Deferred Mappings](Deferred.md#deferred-mappings).

`via` and `inverse` are mutually exclusive. A `via` path is an unqualified
slot name, a `Spec.slot` name, or a fully qualified `lib::Spec.slot` name,
optionally followed by `+` or `*`. An `inverse` reference uses the same name
forms without a path suffix. Empty paths, malformed names, repeated suffixes,
and a Query declaring both forms are export errors.

### Rejected Query Metadata

The following complete definitions are rejected during RDF export. Declaring
both traversal forms is ambiguous:

```xeto
Point : Dict
Equip : Dict {
  points: Query<of:Point, via:"pointRef", inverse:"Point.equips">
}
```

The exporter reports that `fixture.example::Equip.points` cannot declare both
`via` and `inverse`.

An empty traversal does not identify a property:

```xeto
Point : Dict
Equip : Dict {
  points: Query<of:Point, via:"">
}
```

The exporter reports `Empty query path for fixture.example::Equip.points`.
The same rule applies to an empty `inverse` value.

A path may have at most one trailing quantifier:

```xeto
Point : Dict
Equip : Dict {
  points: Query<of:Point, via:"pointRef++">
}
```

The exporter reports an invalid Query path for
`fixture.example::Equip.points`, including the rejected value `pointRef++`.

### Inverse Query and Direct Inverse Property

An inverse reference is resolved in the library that declares the Query. If
the referenced name identifies another Query, that Query must declare `via`;
the exporter reverses its `via` path. If the name identifies an ordinary
property, or no Query declaration with that name is present, the exporter
reverses the named property directly. These two cases are deliberately
distinct:

```xeto
Equip : Dict {
  // Reverses the traversal declared by Point.equips.
  points: Query<of:Point, inverse:"Point.equips">

  // Reverses Point.equipRef directly.
  directPoints: Query<of:Point, inverse:"Point.equipRef">
}

Point : Dict {
  equipRef: Ref<of:Equip>
  equips: Query<of:Equip, via:"equipRef+">
}
```

```turtle
ex:Equip a sh:NodeShape ;
  sh:targetClass ex:Equip ;
  sh:property [
    sh:path [
      sh:oneOrMorePath [ sh:inversePath ex:Point.equipRef ]
    ] ;
  ] ;
  sh:property [
    sh:path [ sh:inversePath ex:Point.equipRef ] ;
  ] .
```

`Equip.points` reverses the referenced Query's `equipRef+` traversal, so its
path retains `sh:oneOrMorePath`. `Equip.directPoints` reverses the ordinary
`Point.equipRef` property directly, so its path has no repetition operator.

A referenced Query without `via` does not contain enough traversal metadata
to invert and is therefore an export error. It is not silently treated as an
ordinary stored property.

## Queries Contributed by Mixins

A query contributed by a mixin may traverse a slot from the contributing
library or a slot from the target spec's library. The path name determines
which RDF predicate SHACL follows.

**Example: `mixin-query-paths`.**

Suppose `base.equips` defines:

```xeto
Equip : Dict {
  basePointRef: Ref<of:Point, maybe>
}

Point : Dict
```

and `app.equips` contributes:

```xeto
+base.equips::Equip {
  pointRef: Ref<of:base.equips::Point, maybe>
  pointsVia: Query<of:base.equips::Point, via:"pointRef">
  pointsViaBase: Query<of:base.equips::Point,
    via:"base.equips::Equip.basePointRef">
  pointsInverse: Query<of:base.equips::Point, inverse:"Point.equips">
}

+base.equips::Point {
  equipRef: Ref<of:base.equips::Equip, maybe>
  equips: Query<of:base.equips::Equip, via:"equipRef">
}
```

The relevant part of the resulting target shape is:

```turtle
base:Equip a sh:NodeShape ;
  sh:targetClass base:Equip ;
  sh:property [
    sh:path app:Equip.pointRef ;
  ] ;
  sh:property [
    sh:path base:Equip.basePointRef ;
  ] ;
  sh:property [
    sh:path [ sh:inversePath app:Point.equipRef ] ;
  ] .
```

For `pointsVia`, the unqualified `pointRef` resolves in the contributing
`app.equips` library, producing `app:Equip.pointRef`. For `pointsViaBase`, the
full name explicitly selects `base:Equip.basePointRef`. This determines which
RDF relationships SHACL follows; the two paths are not interchangeable.

For `pointsInverse`, `Point.equips` first resolves to the query contributed by
`app.equips`. That query traverses `app:Point.equipRef`, so reversing it
produces `[ sh:inversePath app:Point.equipRef ]`.

