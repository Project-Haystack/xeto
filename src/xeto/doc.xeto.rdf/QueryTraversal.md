<!--
title:      Query Traversal
-->

A Xeto Query can follow outgoing references, follow incoming references, or
use the starting record itself. During Result Selection, the SPARQL `SELECT`
uses the translated Query path to reach records.

Translate the Query's `via` or `inverse` path into the `[traversal]` part
of the [SELECT template](QueryResultSelection.md). This part starts at
`?subject` and binds each value reached by the path to `?result`.

Once [Input Validation](QueryInputValidation.md) passes, the executor runs
the SELECT. The path reaches records for the `of` filter to check. The
examples show the SPARQL translation and link to the input checks for each
path.

# Inherited and Overridden Queries

A Query's `of` type and `via` or `inverse` path can be inherited from a parent
type or overridden by a subtype's Query slot of the same name. The following
example shows both cases:

```xeto
Target: Dict { hit }
Narrow: Target { narrow }
Owner: Dict {
  next: Ref?
  alternate: Ref?
  selected: Query<of:Target, via:"next">
}
Inherited: Owner {}
Override: Owner { selected: Query<of:Narrow, via:"alternate+"> }
@start: Override { next: @ordinary, alternate: @narrowRecord }
@ordinary: Target { hit }
@narrowRecord: Narrow { hit, narrow }
```

Query execution takes a starting record and a specific Query definition.
The following [execution requests](QueryExecution.md#execution-request) use the same
starting record but different Query definitions:

| Starting record | Query definition supplied | Result |
| --- | --- | --- |
| `start` | `Inherited.selected` | `ordinary` |
| `start` | `Override.selected` | `narrowRecord` |

For `Owner.selected` and the unchanged `Inherited.selected`, Result Selection
uses the inherited `via:"next"` path and `of:Target` type:

```sparql
SELECT DISTINCT ?result WHERE {
  VALUES ?subject { ex:start }
  ?subject ex:Owner.next ?result .
  ?result a xetoRdf:Record .
  FILTER(isIRI(?result))
  FILTER EXISTS {
    ?result a ?resultType . VALUES ?resultType { ex:Target ex:Narrow }
  }
}
```

For `Override.selected`, Result Selection uses `via:"alternate+"` and
`of:Narrow`:

```sparql
SELECT DISTINCT ?result WHERE {
  VALUES ?subject { ex:start }
  ?subject (ex:Owner.alternate)+ ?result .
  ?result a xetoRdf:Record .
  FILTER(isIRI(?result))
  FILTER EXISTS {
    ?result a ?resultType . VALUES ?resultType { ex:Narrow }
  }
}
```

The requested Query definition determines the path and result type; the
starting record's type does not choose a different Query automatically.
Values not changed by an override remain inherited.

Since `alternate` is inherited from `Owner`, the path still uses
`ex:Owner.alternate`. This example abbreviates the property and class
lists for demonstration.

For global and mixin-contributed fields, resolve property IRIs using the
mappings in [Globals and Covariant Overrides](TypeComposition.md#globals-and-covariant-overrides)
and [Mixins](TypeComposition.md#mixins).

Resolve Query definitions using the [Complete Spec Definition](Foundations.md#complete-spec-definition)
and [Concrete Library Versions](Foundations.md#concrete-library-versions) rules.

# Resolving a Field Name

`via:"next"` follows the `next` field, which can use different property IRIs on
different record types. Include every IRI used for `next` in the resolved
namespace, including inherited and undeclared fields and fields from dependency
libraries. Do not limit the list to types matching `of`.

For example, this Query reaches a Target through a Bridge:

```xeto
Target: Dict {}
Holder: Dict {
  next: Ref?
  deep: Query<of:Target, via:"next+">
}
Bridge: Dict { next: Ref? }
@a: Holder { next: @b }
@b: Bridge { next: @t }
@t: Target {}
```

The two `next` fields produce different RDF properties. The relevant links are:

```turtle
ex:a ex:Holder.next ex:b .
ex:b ex:Bridge.next ex:t .
```

The SPARQL property path combines both alternatives with `|`:

```text
(ex:Holder.next|ex:Bridge.next)
```

Following only `ex:Holder.next` would stop at `b` and miss `t`. Repeating the
combined path with `+` reaches `b` and then `t`; `of:Target` selects only `t`.
These alternatives show the properties used by the example's links; the
resolved set also includes mappings for undeclared `next` fields on other
record types, as required above.

A path uses one property directly or combines several with `|`. The traversal
rules use `P` for that resolved property path; `P` is notation, not a SPARQL
variable.

# Forward and Repeated Traversal

Forward traversal follows the named field. A one-hop path follows one link;
a repeated path follows one or more links:

| Xeto path | Traversal fragment |
| --- | --- |
| `via:"next"` | `?subject P ?result .` |
| `via:"next+"` | `?subject (P)+ ?result .` |

The [worked example](QueryExecution.md#worked-example) declares `Holder.direct` with
`via:"next"` and `Holder.deep` with `via:"next+"`. For `Holder.deep`, the
corresponding SELECT repeats the path with `+`:

```sparql
SELECT DISTINCT ?result WHERE {
  VALUES ?subject { ex:a }
  ?subject (ex:Holder.next)+ ?result .
  ?result a xetoRdf:Record .
  FILTER(isIRI(?result))
  FILTER EXISTS {
    ?result a ?resultType . VALUES ?resultType { ex:Target }
  }
}
```

The example still returns `t`, which has no `next` field to continue through.
Its property alternatives are abbreviated as in the one-hop example. With
additional links, the same clause follows `next` on every reached record,
regardless of whether that intermediate record matches Target.

One-or-more traversal does not include the start merely because the start
fits `of`; a self-link or a cycle can reach it again. Duplicate links and
multiple paths to a record do not duplicate results. Missing referenced records
end their branches without an execution error.

For `via:"next+"`, the SPARQL `+` requires at least one `next` link before a
record can be returned. See [Input Validation](QueryInputValidation.md) for the
SHACL path.

# Inverse Result Selection

An [inverse Query](QuerySlots.md#inverse-query-and-direct-inverse-property) uses the referenced Query's path in reverse and its own `of` type
to select results. The referenced Query's `of` type does not filter the starting record
or intermediate records. No stored triples for computed Query slots are required.

Executable inverse Queries require a fully qualified reference, such as
`example.inverse::Point.equips`. A missing Query or a Query without `via` causes
an execution error. Local names are accepted only by the [schema property-path mapping](QuerySlots.md#inverse-query-and-direct-inverse-property).

| Referenced path | Inverse traversal fragment |
| --- | --- |
| One hop | `?subject ^(P) ?result .` |
| One or more | `?subject (^(P))+ ?result .` |

Both examples use library `example.inverse` and the following `equipRef` links:

```text
p2 (Point) → p1 (Point) → relay1 (Relay) → equip1 (Equipment)
```

## One-Hop Inverse Traversal

A one-hop inverse Query follows one incoming reference. The referenced Query
uses `via:"equipRef"`:

```xeto
Equipment: Dict {
  points: Query<of:Point, inverse:"example.inverse::Point.equips">
}
Point: Dict {
  point
  equipRef: Ref?
  equips: Query<of:Equipment, via:"equipRef">
}
Relay: Dict { equipRef: Ref? }
```

The corresponding SELECT follows incoming links:

```sparql
SELECT DISTINCT ?result WHERE {
  VALUES ?subject { ex:equip1 }
  # Follow one incoming equipRef link.
  ?subject ^(ex:Point.equipRef|ex:Relay.equipRef) ?result .
  ?result a xetoRdf:Record .
  FILTER(isIRI(?result))
  # Select results matching the inverse Query's of:Point.
  FILTER EXISTS {
    ?result a ?resultType . VALUES ?resultType { ex:Point }
  }
}
```

Starting `points` at `equip1` returns no Points: only `relay1` points directly
to `equip1`, and Relay does not match Point.

The one-hop inverse Query still returns no Points with the
[unrelated `equipRef` string](QueryInputValidation.md#inverse-paths).

## Repeated Inverse Traversal

A repeated inverse Query follows one or more incoming references. Change the
referenced Query's path to `via:"equipRef+"`; the inverse declaration stays
the same:

```xeto
Equipment: Dict {
  points: Query<of:Point, inverse:"example.inverse::Point.equips">
}
Point: Dict {
  point
  equipRef: Ref?
  equips: Query<of:Equipment, via:"equipRef+">
}
Relay: Dict { equipRef: Ref? }
@equip1: Equipment {}
@p1: Point { point, equipRef: @relay1 }
@relay1: Relay { equipRef: @equip1 }
@p2: Point { point, equipRef: @p1 }
```

The corresponding SELECT repeats the inverse path with `+`:

```sparql
SELECT DISTINCT ?result WHERE {
  VALUES ?subject { ex:equip1 }
  ?subject (^(ex:Point.equipRef|ex:Relay.equipRef))+ ?result .
  ?result a xetoRdf:Record .
  FILTER(isIRI(?result))
  FILTER EXISTS {
    ?result a ?resultType . VALUES ?resultType { ex:Point }
  }
}
```

Starting `points` at `equip1` returns `p1` and `p2`. Both have a nonempty
`equipRef` path to `equip1`; the Relay is traversed but is not a Point result.

The SELECT examples abbreviate the property and class lists; translations
include every resolved mapping for `equipRef` and every matching Point subtype.
See [Input Validation for inverse paths](QueryInputValidation.md#inverse-paths)
for the SHACL checks that run before the SELECT.

# Identity Traversal

A Query starts at a lookup record. With `via:"id"`, reading the id and looking
it up selects that same record. RDF uses its id as its subject IRI, so the SPARQL translation does not follow
a separate `id` property. Repeating or reversing identity traversal still
reaches the starting record; each identity step counts as a hop.

For example, in library `example.identity`:

```xeto
Target: Dict { hit }
Other: Dict {}
Queries: Dict {
  self: Query<of:Target, via:"id">
  repeatedSelf: Query<of:Target, via:"id+">
  inverseSelf: Query<of:Target, inverse:"example.identity::Queries.self">
  inverseRepeatedSelf: Query<of:Target, inverse:"example.identity::Queries.repeatedSelf">
}
@t: Target { hit }
@outsider: Other {}
```

The instance mapping translates a record's Xeto `id` to the record's RDF
subject IRI. All four Query forms in the example therefore use
`BIND(?subject AS ?result)` in place of a property path. For `self`
starting at `t`, the SELECT is:

```sparql
SELECT DISTINCT ?result WHERE {
  VALUES ?subject { ex:t }
  BIND(?subject AS ?result)
  ?result a xetoRdf:Record .
  FILTER(isIRI(?result))
  FILTER EXISTS {
    ?result a ?resultType . VALUES ?resultType { ex:Target }
  }
}
```

All four Queries return `t` when started at `t`, because `t` matches `of:Target`.
Starting at `outsider` returns no results because Other does not match Target.
For another starting record or Query, bind the supplied starting IRI and use
the Query's own `of` type for Result Selection.

See [Input Validation for identity paths](QueryInputValidation.md#identity-paths)
for the starting-record check and [Result Validation](QueryResultValidation.md)
for the required `hit` marker check.

# Spec-Reference Traversal

The [`spec` field](Mappings.md#the-spec-field) identifies a record's declared Xeto type.
Its Ref can be used in two different operations:

- **Type validation** resolves the Ref to a schema definition and checks the
  record against that definition.
- **Query traversal** reads the Ref and looks for a runtime record with that
  ID. As with other Query paths, that record is returned only if it exists and
  matches the Query's `of` type. The schema definition alone is not a result.

A SPARQL selection query must therefore follow `sys:spec`, which preserves
the exact Ref in the field, rather than `rdf:type`, which may include other
type memberships.
A record without a stored `spec` field has no `sys:spec` value; type membership
alone must not create one.

For example, these definitions belong to library `example.spec`:

```xeto
Base: Dict { hit }
Child: Base {}
Queries: Dict {
  direct: Query<of:Base, via:"spec">
  transitive: Query<of:Base, via:"spec+">
  directInverse: Query<of:Base, inverse:"example.spec::Queries.direct">
  transitiveInverse: Query<of:Base, inverse:"example.spec::Queries.transitive">
}
```

Runtime records may have IDs that equal type names. These runtime records are
shown in Trio; library instance declarations cannot reuse a type's name:

```trio
id: @example.spec::reading
spec: @example.spec::Child
hit
---
id: @example.spec::Child
spec: @example.spec::Base
hit
```

Their instance RDF is:

```turtle
ex:reading a xetoRdf:Record, ex:Child ;
  sys:spec ex:Child ;
  xetoRdf:hasMarker ex:Base.hit .
ex:Child a xetoRdf:Record, ex:Base ;
  sys:spec ex:Base ;
  xetoRdf:hasMarker ex:Base.hit .
```

The traversal property set is exactly `sys:spec`. Direction and repetition
compose with this property as follows:

| Query form | SPARQL path |
| --- | --- |
| `via:"spec"` | `sys:spec` |
| `via:"spec+"` | `(sys:spec)+` |
| Inverse of `via:"spec"` | `^sys:spec` |
| Inverse of `via:"spec+"` | `(^sys:spec)+` |

The complete SELECT combines this path with the starting record and the Query's
result filter, following the [Result Selection structure](QueryResultSelection.md).
Here, `VALUES ?subject` binds the start to `ex:reading`, and `of:Base` selects
records typed as Base or its subtype Child:

```sparql
SELECT DISTINCT ?result WHERE {
  VALUES ?subject { ex:reading }
  ?subject (sys:spec)+ ?result .
  ?result a xetoRdf:Record .
  FILTER(isIRI(?result))
  FILTER EXISTS {
    ?result a ?resultType . VALUES ?resultType { ex:Base ex:Child }
  }
}
```

Starting `direct` at `reading` returns the runtime record named `Child`.
Starting `transitive` there also returns only `Child`: the schema defines
Base, but the instance graph contains no runtime record named Base. A schema
class alone is not a query result. Conversely, inverse traversal from Child
returns reading. Reading's subtype membership makes it eligible for `of:Base`;
it does not create a direct spec reference from reading to Base.

A missing `spec` field ends a forward branch.
The [input shape for `transitive`](QueryInputValidation.md#spec-reference-paths)
shows the checks for `reading`. See
[Instance Data and Schema Vocabulary](QueryResultValidation.md#instance-data-and-schema-vocabulary)
for the graph rules when validating the returned `ex:Child` against Base.
