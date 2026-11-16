# Query Traversal and Input Validation

Input checks execute first, before Result Selection. Each traversal subsection
presents the Xeto declaration and its meaning, then the SPARQL traversal, then
any traversal-specific input checks. This teaching order does not change the
execution order.

## Input Validation

Input Validation uses SHACL to require the starting record to exist and
populated fields followed by the Query to contain references. Input Validation
must pass before Result Selection. For the starting record [`a`](Queries.md#worked-example):

```turtle
[] a sh:NodeShape ;
  sh:targetNode ex:a ;
  sh:nodeKind sh:IRI ;
  sh:property [ sh:path rdf:type ; sh:hasValue sys:Entity ] ;
  sh:or (
    [ sh:property [ sh:path ex:Holder.next ; sh:maxCount 0 ] ]
    [ sh:property [ sh:path sys:hasRefField ; sh:hasValue ex:Holder.next ] ]
  ) ;
  sh:property [ sh:path ex:Holder.next ; sh:nodeKind sh:IRI ] ;
  sh:property [
    sh:path sys:hasMarker ;
    sh:not [ sh:in ( ex:Holder.next ) ]
  ] .
```

The existence check requires an exported record at `ex:a`. The remaining checks
allow an absent field, require reference-field metadata for a populated field,
and reject literal, blank-node, or Marker values. A failed input check is an
execution error; the consumer must not turn it into an empty query result.

The starting record must exist. If the supplied starting ID has no record,
execution must report an error, rather than return an empty result. This differs
from a missing referenced target, which ends only that traversal branch:

| Input | Correct behavior | Incorrect translation |
| --- | --- | --- |
| Starting record `a` is missing | Report an execution error. | Return no results. |
| `a.next` is a Ref to an existing `t: Target` | Return `t`. | Drop `t`. |
| `a.next` is a Ref to a missing record | End that branch without a result. | Report an execution error. |
| `a.next` stores a Unit or named nested Dict | Report an error when trying to follow it. | Follow its IRI or silently return no results. |
| A Ref reaches `t: Target`, but `t` lacks `hit` | Return `t`; report its validation failure separately if validation is requested. | Remove `t` from the results. |

The [reference-field rules](References.md#reference-field-metadata) define the evidence
required for a populated field. Each resolved property gets the checks shown
above, and the traversal construct determines which records receive them:

| Traversal | Reference-check scope |
| --- | --- |
| Forward one-hop reference | Starting record |
| Forward repeated reference | Start and reached records, using a zero-or-more SHACL path |
| Direct inverse reference | Reference evidence is part of SELECT matching, not a global rejection of non-Refs |
| Repeated inverse reference | All candidate records, including disconnected components |
| Identity (`id`, repeated or inverse) | No field checks; starting-record existence still applies |
| Spec reference | Same scope as the corresponding reference traversal, using `sys:spec` |

All forms require the starting record to exist. They do not require it to have
the Query owner's type. Declared Ref cardinality is a record-validation rule
and must not be added as a query execution requirement.

## Traversal

The following sections show how Xeto Query traversal paths translate to SPARQL.

### Inherited and Overridden Queries

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
The following [execution requests](Queries.md#execution-request) use the same
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
  ?result a sys:Entity .
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
  ?result a sys:Entity .
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

### Resolving a Field Name

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

### Forward and Repeated Traversal

Forward traversal follows the named field. A one-hop path follows one link;
a repeated path follows one or more links:

| Xeto path | Traversal fragment | Fields subject to reference checks |
| --- | --- | --- |
| `via:"next"` | `?subject P ?result .` | The starting record's `next` field |
| `via:"next+"` | `?subject (P)+ ?result .` | `next` on the start and every reached record where traversal continues |

The [worked example](Queries.md#worked-example) declares `Holder.direct` with
`via:"next"` and `Holder.deep` with `via:"next+"`. For `Holder.deep`, the
corresponding SELECT repeats the path with `+`:

```sparql
SELECT DISTINCT ?result WHERE {
  VALUES ?subject { ex:a }
  ?subject (ex:Holder.next)+ ?result .
  ?result a sys:Entity .
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

Input Validation uses a **zero-or-more** path to apply the
[reference checks](#input-validation) to the starting record and every
reached record. Result traversal uses **one-or-more** because `via:"next+"`
requires at least one hop. The starting record is returned only if a self-link
or cycle reaches the starting record again and the record matches `of`.
Forward traversal does not check unreachable records.

### Inverse Result Selection

An [inverse Query](QuerySlots.md#inverse-query-and-direct-inverse-property) uses the referenced Query's path in reverse and its own `of` type
to select results. The referenced Query's `of` type does not filter the starting record
or intermediate records. No stored triples for computed Query slots are required.

Executable inverse Queries require a fully qualified reference, such as
`example.inverse::Point.equips`. A missing Query or a Query without `via` causes
an execution error. Local names are accepted only by the [schema property-path mapping](QuerySlots.md#inverse-query-and-direct-inverse-property).

Both examples use library `example.inverse` and the following `equipRef` links:

```text
p2 (Point) → p1 (Point) → relay1 (Relay) → equip1 (Equipment)
```

#### One-Hop Inverse Traversal

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

The corresponding SELECT checks the incoming link and its Ref metadata:

```sparql
SELECT DISTINCT ?result WHERE {
  VALUES ?subject { ex:equip1 }
  # Follow one incoming equipRef link.
  ?subject ^(ex:Point.equipRef|ex:Relay.equipRef) ?result .
  ?result a sys:Entity .
  FILTER(isIRI(?result))
  # Require Ref metadata for the property containing that link.
  FILTER EXISTS {
    VALUES ?referenceProperty { ex:Point.equipRef ex:Relay.equipRef }
    ?result sys:hasRefField ?referenceProperty .
    ?result ?referenceProperty ?subject .
  }
  # Select results matching the inverse Query's of:Point.
  FILTER EXISTS {
    ?result a ?resultType . VALUES ?resultType { ex:Point }
  }
}
```

Starting `points` at `equip1` returns no Points: only `relay1` points directly
to `equip1`, and Relay does not match Point.

An incoming RDF link alone is insufficient: a non-Ref value can export to the
same IRI. One-hop inverse matching accepts only Ref values, so Result Selection
checks the link's [reference-field metadata](References.md#reference-field-metadata).
Using `?referenceProperty` in both triples ensures that the Ref metadata
describes the field containing the incoming link. A Ref in another field does
not make the link eligible.

#### Repeated Inverse Traversal

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
  ?result a sys:Entity .
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

#### Input Validation Differences

One-hop inverse traversal checks Ref metadata during Result Selection.
Repeated inverse traversal checks candidate records during Input Validation:

| Referenced path | Inverse traversal fragment | Reference handling |
| --- | --- | --- |
| One hop | `?subject ^(P) ?result .` | Require reference evidence on each matching edge; ignore non-Ref candidates. |
| One or more | `?subject (^(P))+ ?result .` | Check the followed field throughout the candidate record set, including disconnected records. |

For transitive inverse traversal over ordinary reference fields, reference
checks include records carrying the followed tag in disconnected parts of the
graph, even when they cannot contribute a result. This preserves native Xeto's
execution-error behavior: a non-Ref in such a field can cause the query to fail.
Both forms require an existing starting record. One-hop inverse matching does
not reject non-Ref fields merely because they are present elsewhere in the graph.

For example, add this runtime record to either example's input:

```trio
id: @example.inverse::unrelated
equipRef: "not a Ref"
```

This record has no path to `equip1`, but its `equipRef` field causes the
repeated inverse Query to fail its reference checks. The one-hop inverse
Query ignores this non-Ref value and still returns no Points.

### Identity Traversal

A Query with `via:"id"` selects the starting record itself. RDF represents the
record's identity as its subject IRI, so the SPARQL translation does not follow
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
  ?result a sys:Entity .
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

[Input Validation](#input-validation) checks only that the starting record
exists. Identity traversal needs no field-property alternatives or Ref
metadata, and does not inspect other fields or disconnected records.

[Result Validation](QueryResultValidation.md#query-result-validation) checks Target's required `hit` marker
as usual. Removing `hit` leaves `t` in Result Selection but causes Result
Validation to fail.

### Spec-Reference Traversal

The [`spec` field](Mappings.md#the-spec-field) identifies a record's declared Xeto type.
Its Ref can be used in two different operations:

- **Type validation** resolves the Ref to a schema definition and checks the
  record against that definition.
- **Query traversal** reads the Ref and looks for a runtime record with that
  ID. As with other Query paths, that record is returned only if it exists and
  matches the Query's `of` type. The schema definition alone is not a result.

A SPARQL selection query must therefore follow `sys:spec`, which preserves
the exact Ref in the field, rather than `rdf:type`, which may include other
type memberships. The instance must also have `sys:hasRefField sys:spec`,
identifying the field as a Ref as described in
[Reference-field metadata](References.md#reference-field-metadata).
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
ex:reading a sys:Entity, ex:Child ;
  sys:spec ex:Child ;
  sys:hasRefField sys:spec ;
  sys:hasMarker ex:Base.hit .
ex:Child a sys:Entity, ex:Base ;
  sys:spec ex:Base ;
  sys:hasRefField sys:spec ;
  sys:hasMarker ex:Base.hit .
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
result filter, following the [Result Selection structure](Queries.md#result-selection).
Here, `VALUES ?subject` binds the start to `ex:reading`, and `of:Base` selects
records typed as Base or its subtype Child:

```sparql
SELECT DISTINCT ?result WHERE {
  VALUES ?subject { ex:reading }
  ?subject (sys:spec)+ ?result .
  ?result a sys:Entity .
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

Traversal through `spec` uses the same reference checks as ordinary fields,
with `sys:spec` as the property being checked. For forward traversal, a missing
field ends the branch; a present non-Ref causes an execution error. Inverse
traversal follows the checks in [Inverse Result Selection](#inverse-result-selection).

For `transitive` starting at `reading`, the input shape combines three checks:

| Shape component | Requirement |
| --- | --- |
| `sh:targetNode`, `sh:nodeKind`, and the `rdf:type` constraint | `reading` must be an IRI identifying an exported record (`sys:Entity`). |
| `sh:zeroOrMorePath sys:spec` | Apply the field checks to the start and every node reached through `sys:spec`. |
| The nested `sh:node` | Allow an absent `spec` field; otherwise require Ref evidence and IRI values. Reject a Marker-valued `spec`, which is represented through `sys:hasMarker`. |

Together, these produce:

```turtle
[] a sh:NodeShape ;
  sh:targetNode ex:reading ;
  sh:nodeKind sh:IRI ;
  sh:property [ sh:path rdf:type ; sh:hasValue sys:Entity ] ;
  sh:property [
    sh:path [ sh:zeroOrMorePath sys:spec ] ;
    sh:node [
      sh:or (
        [ sh:property [ sh:path sys:spec ; sh:maxCount 0 ] ]
        [ sh:property [ sh:path sys:hasRefField ; sh:hasValue sys:spec ] ]
      ) ;
      sh:property [ sh:path sys:spec ; sh:nodeKind sh:IRI ] ;
      sh:property [ sh:path sys:hasMarker ; sh:not [ sh:in ( sys:spec ) ] ]
    ]
  ] .
```

Result Selection still requires one or more steps. Result Validation checks
the returned Child record against Base, by targeting `ex:Child` with Base's
reusable constraints. Schema triples must not supply runtime fields when an
IRI is shared; see [Instance Data and Schema Vocabulary](QueryResultValidation.md#instance-data-and-schema-vocabulary).
