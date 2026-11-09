# Query Execution and Result Validation

This section defines how to execute Xeto Queries over exported RDF using
SPARQL, with SHACL checks before and after Result Selection. SPARQL follows the
Query's path and selects records matching its `of` type. SHACL checks whether execution
can proceed and, when requested, validates the returned records. Execution
errors and validation failures are reported separately.

The translation has three stages: Input Validation, Result Selection, and
optional Result Validation:

```text
Input Validation (SHACL)
    starting-record existence
    + reference checks required by traversal

Result Selection (SPARQL)
    starting-record binding
    + traversal
    + record restriction
    + result-matching predicate
    + unique-result projection

Result Validation (SHACL, when requested)
    target the returned identities
    + apply the Query's of constraints
```

Input Validation checks conditions that native Xeto checks during execution.
Failed Input Validation returns an execution error rather than an empty result.
Records that fail Result Validation remain in the results, with validation
failures reported alongside them. The [schema property-path mapping](QuerySlots.md#shacl-validation)
describes navigation but does not execute these stages.

Use the same [resolved library versions](Foundations.md#concrete-library-versions) as the RDF export. Apply the
[conformance error rule](Overview.md#conformance) when a Query cannot be translated faithfully.

## Execution Request

A Query begins with a request to a query executor. This **execution request**
supplies:

| Input | Meaning |
| --- | --- |
| Query definition | The specific Query slot to execute, such as `ex::Holder.direct`. |
| Starting record | The RDF IRI of the record from which traversal starts. |
| Instance graph | The records to search, exported using the [instance profile](Overview.md#overview). |
| [Resolved namespace](Foundations.md#namespace) | The library definitions and concrete versions used for the instance export, including active mixins. |
| Validate results | Boolean: run Result Validation when `true`; skip it when `false`. |

## Worked Example

The following example shows how Result Selection and Result Validation
translate a Xeto Query:

```xeto
Target: Dict { hit }
Holder: Dict {
  next: MultiRef?
  direct: Query<of:Target, via:"next">
  deep: Query<of:Target, via:"next+">
}
@a: Holder { next: MultiRef { @t } }
@t: Target { hit }
```

Result Selection returns `t` when `direct` starts at `a`. Result Validation
checks that `t` has the required `hit` marker. A record declared as Target can
still be missing that marker.

The records to be searched must first be exported to RDF using the instance
profile. Run the SELECT against the resulting RDF. Result Selection must run
using only triples produced by the instance export. Do not include schema
triples or add extra triples derived from schema relationships.

The same example exports to the following RDF triples. (Not a full export; only
the triples needed for demonstration are shown):

In these examples, `ex:` denotes the example library's concrete versioned
namespace, and `sys:` denotes the resolved sys library namespace. Standard
prefixes follow [Namespaces and IRIs](Foundations.md#namespaces-and-iris).

```turtle
ex:a a sys:Entity, ex:Holder ;
  ex:Holder.next ex:t ;
  sys:hasRefField ex:Holder.next .
ex:t a sys:Entity, ex:Target ;
  sys:hasMarker ex:Target.hit .
```

The three stages use these triples as follows:

| Stage | Operation on the exported RDF |
| --- | --- |
| [Input Validation](#input-validation) | SHACL checks that `ex:a` exists and that `next` contains references, using `sys:hasRefField` and the reference values. |
| [Result Selection](#result-selection) | SPARQL follows `ex:Holder.next` and returns `ex:t` because `ex:t` has type `ex:Target`. |
| [Result Validation](#result-validation) | SHACL checks that `ex:t` has the required `sys:hasMarker ex:Target.hit` statement. |

## Result Selection

For `Holder.direct` in the [worked example](Queries.md#worked-example), Result
Selection uses the following SELECT. Property and class lists are abbreviated
for demonstration:

```sparql
SELECT DISTINCT ?result WHERE {             # Unique-result projection
  VALUES ?subject { ex:a }                  # Starting-record binding
  ?subject ex:Holder.next ?result .         # Traversal
  ?result a sys:Entity .                    # Record restriction
  FILTER(isIRI(?result))
  FILTER EXISTS {                          # Result matching
    ?result a ?resultType . VALUES ?resultType { ex:Target }
  }
}
```

The query returns `ex:t`, which is reached through `ex:Holder.next` and has
type `ex:Target`.

Each part has one responsibility:

| Part | Translation rule | What supplies its contents? |
| --- | --- | --- |
| Starting-record binding | Bind `?subject` to the supplied record IRI, using an RDF-term binding or a safely serialized `VALUES` clause. | The execution request |
| Traversal | Relate `?subject` to `?result`, or bind them to the same identity for `id`. | The Query's path, direction, repetition, and resolved field properties |
| Record restriction | Require `?result a sys:Entity` and `isIRI(?result)`. | Fixed structure; the sys namespace uses its resolved version |
| Result matching | Require the reached record to fit the Query's `of` type. | The target type and resolved schemas |
| Unique-result projection | Return `SELECT DISTINCT ?result`, with no ordering guarantee. | Fixed structure |

The pieces compose as follows. This is a translation template, not executable
SPARQL; the bracketed parts stand for the constructs defined below.

```text
SELECT DISTINCT ?result WHERE {
  [starting-record binding]
  [traversal]
  ?result a sys:Entity .
  FILTER(isIRI(?result))
  FILTER([result-matching predicate])
}
```

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

A path uses one property directly or combines several with `|`. The rules
below write `P` for that resolved property path; `P` is notation, not a SPARQL
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

[Result Validation](#result-validation) checks Target's required `hit` marker
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
result filter, following the [Result Selection structure](#result-selection).
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
IRI is shared; see [Instance Data and Schema Vocabulary](#instance-data-and-schema-vocabulary).

## Result Matching

Result matching decides which records reached by traversal are returned,
using the Query's `of` type. For an ordinary named type such as `Target`, a
reached record matches when its declared type is Target or a subtype of Target.
Required fields are checked separately during Result Validation.

Sugar types also allow a reached record to match based on the record's field
values, even when the record's declared type is not a subtype of the sugar type.
Non-sugar unions and intersections use type membership. The rules below define
how each target forms the SELECT's result-matching predicate.

### Type Membership

A record matches a target type when its declared type is the target itself or
one of its subtypes. The SPARQL query checks this by comparing the record's
declared RDF type with the matching class IRIs:

```sparql
FILTER EXISTS {
  ?result a ?type .
  VALUES ?type { [matching class IRIs] }
}
```

The list includes all matching classes in the resolved namespace, including
ordinary and computed sugar subtypes. This check uses the record's declared
type; it does not inspect required fields.

The rules below refer to this check as `type_match(T)`, where `T` is the target
type. In the opening Target example, `type_match(Target)` is the complete
result predicate.

### Sugar Matching

For a sugar target `S`, `anchor(S)` means its underlying named, non-sugar type.
For example, the anchor of `Sweet: Base <sugar>` is Base. Matching has two
alternatives:

```text
sugar_match(S)
    = type_match(S)
      OR (
        type_match(anchor(S))
        AND present(required tag)
        AND equal(invariant tag, expected value)
        AND ...
      )
```

The predicate names above stand for conditions in the SELECT's `FILTER`.
If the sugar has no required markers or invariants, its structural alternative
checks only the anchor's type.

For `present(tag)` and `equal(tag, value)`, resolve the tag to its property IRIs
using the [field-name rule](#resolving-a-field-name). Then apply the corresponding
test below:

| Predicate | Translation rule |
| --- | --- |
| `present(tag)` | An `EXISTS` test for any value of a property representing the tag, OR an `EXISTS` test for that property's IRI as a `sys:hasMarker` or `sys:hasRefField` value. Reference-field metadata preserves presence for an explicitly stored empty MultiRef. |
| `equal(tag, value)` | An `EXISTS` test for a property representing the tag whose value equals the invariant under Xeto's value-equality rules. |

Each `EXISTS` checks the current `?result` for a matching field. It uses the
resolved property IRIs without following references or selecting other records.
The example below shows the presence tests;
[Invariant Values](#invariant-values) shows the equality test.

For example, consider these definitions. The illustrative library name is
`example.fitting`, and `ex:` denotes its versioned RDF namespace:

```xeto
Base: Dict { required, *hot: Marker, *extra: Marker, *label: Str }
Sweet: Base <sugar> { hot, label: "default" }
Queries: Dict {
  sweet: Query<of:Sweet, via:"id">
}
@candidate: Base { required, hot }
```

The relevant instance RDF is:

```turtle
ex:candidate a sys:Entity, ex:Base ;
  sys:hasMarker ex:Base.required, ex:Base.hot .
```

`candidate` is declared as `Base`, not `Sweet`. It still matches `of:Sweet`
because its type matches Sweet's underlying named type, or **nominal anchor**,
`Base`, and it has the required `hot` field. The default `label` value is not
a Result Selection requirement.

The `via:"id"` path selects the starting record itself. Its SPARQL translation
uses `BIND(?subject AS ?result)` rather than following a stored property.
The result predicate has two alternatives:

- The record's type fits `Sweet`.
- The record's type fits `Base`, and its `hot` field is present.

Class and property lists are abbreviated to the relevant definitions. A
translation must include all applicable alternatives in the current namespace.
The SPARQL selection query is:

```sparql
SELECT DISTINCT ?result WHERE {
  VALUES ?subject { ex:candidate }
  BIND(?subject AS ?result)
  ?result a sys:Entity .
  FILTER(isIRI(?result))
  FILTER(
    EXISTS { ?result a ?type . VALUES ?type { ex:Sweet } }
    || (
      EXISTS { ?result a ?anchor . VALUES ?anchor { ex:Base ex:Sweet } }
      && (
        EXISTS {
          VALUES ?property { ex:Base.hot ex:Sweet.hot }
          ?result ?property ?value .
        }
        || EXISTS {
          VALUES ?property { ex:Base.hot ex:Sweet.hot }
          ?result sys:hasMarker ?property .
        }
        || EXISTS {
          VALUES ?property { ex:Base.hot ex:Sweet.hot }
          ?result sys:hasRefField ?property .
        }
      )
    )
  )
}
```

The last three `EXISTS` blocks check whether `hot` is present:

- `?result ?property ?value` checks for a field stored as an ordinary value.
- `?result sys:hasMarker ?property` checks for a field stored as a Marker.
- `?result sys:hasRefField ?property` also recognizes an explicitly stored empty
  MultiRef, which has no ordinary property values.

The Marker check matches `ex:candidate`, so the second alternative selects it.

Because `hot` is a Marker, valid RDF stores it using `sys:hasMarker`. The
ordinary-value check also selects records with an invalid value such as
`hot: "oops"`, matching Xeto's presence check. Result Validation rejects the invalid
value, as shown below.

#### Result Selection and Result Validation

A runtime record with `hot` but no `required` marker has this relevant RDF:

```turtle
ex:candidate a sys:Entity, ex:Base ;
  sys:hasMarker ex:Base.hot .
```

The SPARQL selection query is unchanged:

| Operation | Outcome |
| --- | --- |
| Result Selection with `of:Sweet` | Return `candidate`: `hot` is still present. |
| Result Validation against `Sweet` | Fail: the inherited `required` marker is missing. |

Sugar's structural alternative checks its required markers and invariant
values, including requirements inherited from other sugar definitions. It does
not check defaults or ordinary required fields inherited from its nominal
anchor. A required marker checks field presence, not the stored value's type.
Result Validation checks those additional requirements.

Runtime records can be missing required fields. Each table row represents a
separate starting record with an `id`; the other fields use Trio notation,
separated by semicolons.

| Runtime fields | Selected by `sweet`? | Valid against Sweet? |
| --- | --- | --- |
| `spec: @example.fitting::Base`; `required`; `hot` | Yes | Yes |
| `spec: @example.fitting::Base`; `hot` | Yes | No: `required` is missing |
| `spec: @example.fitting::Base`; `required`; `hot: "oops"` | Yes | No: `hot` is not a Marker |
| `spec: @example.fitting::Base`; `required`; `hot`; `label: "different"` | Yes | Yes: the default does not require this value to equal `"default"` |
| `spec: @example.fitting::Base`; `required` | No | No: `hot` is missing |
| `spec: @example.fitting::Sweet`; `required` | Yes: its declared type matches | No: `hot` is missing |

#### Computed Sugar Subtyping

A sugar type can fit another sugar type without a declared inheritance link
when they have the same anchor and its requirements include all of the target's
requirements. For example:

```xeto
MoreSweet: Base <sugar> { hot, extra }
```

`MoreSweet` does not declare `Sweet` as its base. It nevertheless fits `Sweet`:
both have the anchor `Base`, and MoreSweet's requirements include Sweet's
required `hot` marker. The `label` default does not participate in this comparison.

The type-membership alternative for `of:Sweet` must therefore include both
classes. For the runtime record shown below, the complete SELECT is:

```sparql
SELECT DISTINCT ?result WHERE {
  VALUES ?subject { ex:typedCandidate }
  BIND(?subject AS ?result)
  ?result a sys:Entity .
  FILTER(isIRI(?result))
  FILTER(
    EXISTS { ?result a ?type . VALUES ?type { ex:Sweet ex:MoreSweet } }
    || (
      EXISTS { ?result a ?anchor . VALUES ?anchor { ex:Base ex:Sweet ex:MoreSweet } }
      && (
        EXISTS {
          VALUES ?property { ex:Base.hot ex:Sweet.hot ex:MoreSweet.hot }
          ?result ?property ?value .
        }
        || EXISTS {
          VALUES ?property { ex:Base.hot ex:Sweet.hot ex:MoreSweet.hot }
          ?result sys:hasMarker ?property .
        }
        || EXISTS {
          VALUES ?property { ex:Base.hot ex:Sweet.hot ex:MoreSweet.hot }
          ?result sys:hasRefField ?property .
        }
      )
    )
  )
}
```

For example, this runtime record declares `MoreSweet` but has no markers:

```trio
id: @example.fitting::typedCandidate
spec: @example.fitting::MoreSweet
```

Its relevant RDF is:

```turtle
ex:typedCandidate a sys:Entity, ex:MoreSweet .
```

It matches the type-membership alternative for `of:Sweet`, then fails
Result Validation because the required markers are missing. In
general, computed sugar subtyping requires a compatible anchor and every target
requirement with the same required value. The sugar subtype may impose additional
requirements, as `MoreSweet` does with `extra`.

#### Invariant Values

An invariant, unlike a default, adds a value-equality test to sugar's structural
alternative. For example:

```xeto
Named: Base <sugar> {
  label: Str <invariant> "wanted"
}
```

For records declared as `Base`, the structural alternative for `of:Named`
requires `label` to equal `"wanted"`. If the opening candidate is queried
against Named, the complete SELECT is:

```sparql
SELECT DISTINCT ?result WHERE {
  VALUES ?subject { ex:candidate }
  BIND(?subject AS ?result)
  ?result a sys:Entity .
  FILTER(isIRI(?result))
  FILTER(
    EXISTS { ?result a ?type . VALUES ?type { ex:Named } }
    || (
      EXISTS { ?result a ?anchor . VALUES ?anchor { ex:Base ex:Sweet ex:MoreSweet ex:Named } }
      && EXISTS {
        VALUES ?property { ex:Base.label ex:Named.label }
        ?result ?property ?label .
        # The invariant requires label to equal "wanted".
        FILTER(?label = "wanted")
      }
    )
  )
}
```

The candidate `ex:candidate` has no `label`, so this query returns no result. Adding
`label: "wanted"` makes the structural alternative match.
As with `Sweet`, a declared type that fits `Named` can match the separate
type-membership alternative without running the structural tests.

Invariant comparisons must use Xeto's value-equality rules for every supported
representation. For example, an invariant requiring integer `2` compares the
integer value, not the spelling of the RDF literal: `"2"^^xsd:integer` and
`"02"^^xsd:integer` both represent that value.

If the translator cannot express an invariant's comparison using those rules,
it must report a translation error. Skipping the comparison would allow records
that do not satisfy the invariant to match.

### Union Targets

A union target selects records whose declared type fits at least one member.
For example:

```xeto
Other: Dict { other }
BaseOrOther: Base | Other
```

A record declared as `Base` or `Other` matches `of:BaseOrOther`. Result Selection
checks its declared type; Result Validation checks required fields.

A union can also include a sugar member:

```xeto
Either: Sweet | Other
CompoundQueries: Dict {
  either: Query<of:Either, via:"id">
}
```

With `candidate` declared as `Base` and containing `required` and `hot`, the
two Query result types produce different Result Selection outcomes:

| Query result type | Selected? |
| --- | --- |
| `Sweet` | Yes: Base membership and the `hot` field satisfy sugar's structural alternative. |
| `Either` | No: Base does not fit either union member by type membership. |

Including `Sweet` does not enable its structural matching alternative: the
union still checks type membership. The identity binding, Entity check,
and IRI check stay the same; only the result predicate changes:

```sparql
SELECT DISTINCT ?result WHERE {
  VALUES ?subject { ex:candidate }
  BIND(?subject AS ?result)
  ?result a sys:Entity .
  FILTER(isIRI(?result))
  FILTER EXISTS {
    ?result a ?type .
    VALUES ?type { ex:Sweet ex:MoreSweet ex:Other ex:Either }
  }
}
```

The list includes computed sugar subtypes such as `MoreSweet`, but no structural
`hot` test. Other qualifying types in the current namespace must also be included.

### Intersection Targets

An ordinary intersection target selects records whose declared type is the
intersection's type or a subtype of that intersection. Intersections are
nominally typed by default: two separately named intersections with the same
members are not interchangeable. For example:

```xeto
Both: Base & Other
IntersectionQueries: Dict {
  both: Query<of:Both, via:"id">
}
```

The `both` Query targets the intersection type `Both`. The following runtime
records have the same marker names but different declared types:

```trio
id: @example.fitting::baseRecord
spec: @example.fitting::Base
required
other
---
id: @example.fitting::bothRecord
spec: @example.fitting::Both
required
other
```

Their relevant RDF is:

```turtle
ex:baseRecord a sys:Entity, ex:Base ;
  sys:hasMarker ex:Base.required, ex:Base.other .
ex:bothRecord a sys:Entity, ex:Both ;
  sys:hasMarker ex:Base.required, ex:Other.other .
```

| Starting record | Selected by `both`? |
| --- | --- |
| `baseRecord` | No: having both fields does not make its declared type fit Both. |
| `bothRecord` | Yes: its declared type is Both. |

Starting at `bothRecord`, the complete SELECT tests the eligible classes,
not the fields:

```sparql
SELECT DISTINCT ?result WHERE {
  VALUES ?subject { ex:bothRecord }
  BIND(?subject AS ?result)
  ?result a sys:Entity .
  FILTER(isIRI(?result))
  FILTER EXISTS {
    ?result a ?type .
    VALUES ?type { ex:Both }
  }
}
```

If the intersection itself resolves to a sugar type, Result Selection uses the
[sugar matching rule](#sugar-matching) instead. Its nominal types must reduce
to one most specific anchor. The other member can narrow the anchor to a
subtype; it need not be the original anchor itself:

```xeto
Derived: Base {}
NarrowSweet: Sweet & Derived
```

`NarrowSweet` has anchor Derived and requires Sweet's `hot` marker. In contrast,
`Sweet & Other` is a spec compilation error: Base and Other are unrelated
nominal anchors. This restriction is checked before record validation; see
[Nominal Anchor](doc.xeto::Sugar#nominal-anchor).

For `of:NarrowSweet`, replace the final `FILTER EXISTS` in the SELECT above
with this filter. The type-membership branch now checks for types that fit
NarrowSweet; the added `||` branch requires both Derived membership and the
presence of `hot`:

```sparql
FILTER(
  EXISTS {
    ?result a ?type .
    VALUES ?type { ex:NarrowSweet }
  }
  || (
    EXISTS {
      ?result a ?anchor .
      VALUES ?anchor { ex:Derived ex:NarrowSweet }
    }
    && (
      # Preserve presence matching even for a malformed non-Marker value.
      EXISTS {
        VALUES ?property { ex:Derived.hot ex:NarrowSweet.hot }
        ?result ?property ?value .
      }
      || EXISTS {
        # A valid hot Marker uses sys:hasMarker.
        VALUES ?property { ex:Derived.hot ex:NarrowSweet.hot }
        ?result sys:hasMarker ?property .
      }
      || EXISTS {
        VALUES ?property { ex:Derived.hot ex:NarrowSweet.hot }
        ?result sys:hasRefField ?property .
      }
    )
  )
)
```

The class and property lists are abbreviated; a translation must include the
full resolved sets. `bothRecord` does not fit the Derived anchor, so adding
`hot` alone would not make it match. A record declared as `Derived` with `hot`
matches the structural branch. The `||` separates the two ways to match the
sugar target; the `&&` requires its anchor and field constraints together.

### Records Declared with Compound Types

The previous examples use a compound as the Query's target type. A record's
declared type can also be a compound. In that case, Result Selection depends on
what the declared type guarantees:

- `Both: Base & Other` guarantees membership in both Base and Other. A record
  declared as `Both` therefore matches a Query targeting either type. It also
  matches `Either: Sweet | Other` through its Other member.
- `Either: Sweet | Other` guarantees membership in one alternative, but does
  not say which one. A record declared as `Either` cannot match a Query targeting
  Base by type membership: Sweet fits Base, but Other does not.

An intersection's declared type can therefore fit a target through one member;
a union's declared type fits a target only if every alternative fits it.

Ordinary inheritance still applies. For example:

```xeto
EitherAlias: Either
```

A record declared as `EitherAlias` matches `of:Either` because Either is its
base type. The reverse does not follow: a record declared as `Other` matches
`of:Either`, but does not match `of:EitherAlias`. Inheriting from a union does
not make the new type another union of the same members.

Apply these membership rules recursively to nested compounds. Use the type
relationships to determine the matching RDF classes; a record's fields do not
establish type membership.

### Choice Targets

Choice targets select records by type membership. For example:

```xeto
Flavor: Choice
Vanilla: Flavor { vanilla }
```

This runtime record declares Vanilla but omits its required marker:

```trio
id: @example.fitting::vanilla
spec: @example.fitting::Vanilla
```

Its declared type matches both Flavor and Vanilla. Result Validation checks the
Query's target type, so the missing marker produces different Result Validation
outcomes:

| Query target | Selected? | Valid against the target? |
| --- | --- | --- |
| `Flavor` | Yes | Yes: Flavor does not require the `vanilla` marker. |
| `Vanilla` | Yes | No: Vanilla requires the `vanilla` marker. |

### Scalar Targets

Scalar targets do not match ordinary records. However, a malformed runtime
record can declare a scalar type:

```trio
id: @invalid
spec: @sys::Str
```

If traversal reaches this record, native Xeto selects it for `of:Str` because
its declared type matches. The record still fails validation: declaring Str
does not make a record a string.

A translator that cannot support Result Validation for scalar targets must
reject the Query. Returning an empty result would disagree with native Query
execution for records such as `invalid`.

## Validation

Input Validation asks whether execution can proceed. Query-Body Validation
checks how many selected records match each required or optional member.
Result Validation asks whether the selected records conform to `of`.
These checks have different targets and must remain separate; a matching
record's invalid contents are not an input error unless traversal itself
encounters an invalid reference value.

### Input Validation

Input Validation uses SHACL to require the starting record to exist and
populated fields followed by the Query to contain references. Input Validation
must pass before Result Selection. For the starting record `a`:

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

### Query-Body Validation

A Query body declares requirements on the result set. Each declaration inside
the braces is a **body member**. For example, this Query has two body members,
`temperatureSensor` and `pressureSensor`:

```xeto
Point: Dict {}
Temperature: Point { temperature }
Pressure: Point { pressure }
Equipment: Dict {
  pointRefs: MultiRef?
  points: Query<of:Point, via:"pointRefs"> {
    temperatureSensor: Temperature
    pressureSensor: Pressure?
  }
}
@equip: Equipment { pointRefs: MultiRef { @t, @p, @other } }
@t: Temperature { temperature }
@p: Pressure { pressure }
@other: Point {}
```

Starting `points` at `equip` selects `t`, `p`, and `other`.
The Query body adds requirements on that result set:

| Xeto declaration | Meaning |
| --- | --- |
| `of:Point, via:"pointRefs"` | Select reached records that fit Point. |
| `temperatureSensor: Temperature` | Require exactly one Temperature match among the results. |
| `pressureSensor: Pressure?` | Allow zero or one Pressure match among the results. |

The names `temperatureSensor` and `pressureSensor` identify requirements;
the selected records do not need fields with those names. Removing `t` from
`pointRefs` leaves `temperatureSensor` missing. Adding another Temperature
record makes that requirement ambiguous. Removing `p` is valid because
`pressureSensor` is optional. The additional Point `other` remains a Query
result even though it satisfies neither requirement.

Each body member counts distinct matching records independently. Repeating a
Ref does not increase the count. One record can satisfy two members if the
record fits both requirements. Query-Body Validation does not assign records
to names as `queryNamed` does, or establish that matching records pass Result
Validation.

#### Composing Body Validation

Query-Body Validation reuses Result Selection's traversal and `of`
matching. The checks on each *body member* are described by the
following rules. Let `R` be the Query's result set, and let `M` be one body-member
declaration, such as `temperatureSensor: Temperature`.

```text
Result Selection:
    starting record + traversal + of matching → result set R

Query-Body Validation, for each declaration M:
    R + member matching and claim rules → matches(R, M)
    matches(R, M) → temporary RDF links
    temporary RDF links + required/optional count → SHACL verdict
```

Input Validation runs before Result Selection. Inverse and repeated paths
change how the Query obtains its result set `R`; they do not change how body
members match and count those results.

A result's `spec` field can name an ordinary type or a particular body member.
This determines whether the record is **free** or **claimed**.

In the sensor example, a record with the following `spec` is **free**: it does
not name a body member. Query-Body Validation considers it for each member whose
requirements it fits:

```trio
spec: @example.body::Temperature
```

A record with this `spec` is **claimed** by `temperatureSensor` and counts
only toward that member:

```trio
spec: @example.body::Equipment.points.temperatureSensor
```

Xeto calls this explicit association **anchoring**; see
[Query Constraints](doc.xeto::Constraints#query-constraints).
Creating the second record requires knowing the body member's qualified name.
Ordinary records do not need that knowledge. Matching free records uses the
existing [result-matching rules](#result-matching), including sugar constraints.

To identify a record's claim, start at its declared type and follow its nominal
base chain. The first body member encountered is the one it claims. Matching
a member through sugar's structural rules does not create a claim.

For each body member, count its claimed records if any are present in `R`.
Otherwise, count the free records that fit it. A claimed record never counts
toward a different member.

```text
claims(R, M) = records in R whose declared type claims member M
free(R)      = records in R that claim no member of this Query body

matches(R, M) = claims(R, M), if claims(R, M) is nonempty
                otherwise records in free(R) that fit member M
```

A claimed record still counts if it is missing required fields; a matching
free record does not replace it. Result Validation checks record fields separately.

#### Representing Matches for SHACL

Once the matching records for each body member are known, SHACL needs RDF
links it can count. A SPARQL `CONSTRUCT` creates these links: where a `SELECT`
returns rows, a `CONSTRUCT` produces triples using the bindings from its
`WHERE` clause. For Query-Body Validation, each triple links the starting record
to a matching result through the body member's IRI:

```sparql
CONSTRUCT { ?subject ?requirement ?result }
```

Here, `?subject` is the starting record, `?requirement` is the body member's
IRI, and `?result` is one record in `matches(R, M)`. The template does not
introduce new traversal or fitting rules. The `WHERE` clause combines the
existing Result Selection with the member-matching and claim rules above.

For the sensor example, the temporary graph contains:

```turtle
ex:equip ex:Equipment.points.temperatureSensor ex:t ;
         ex:Equipment.points.pressureSensor ex:p .
```

There is no link for `other`, because that result matches neither member.
These triples are validation evidence. The temporary graph must remain
separate from stored instances.

Each body-member declaration contributes one SHACL property constraint.
The required/optional distinction changes only the minimum count:

```turtle
[] a sh:NodeShape ;
  sh:targetNode ex:equip ;
  sh:property [
    sh:path ex:Equipment.points.temperatureSensor ;
    sh:minCount 1 ;
    sh:maxCount 1
  ] ;
  sh:property [
    sh:path ex:Equipment.points.pressureSensor ;
    sh:minCount 0 ;
    sh:maxCount 1
  ] .
```

A minimum-count violation corresponds to `sys::missingQuery`; a maximum-count
violation corresponds to `sys::ambiguousQuery`. The report identifies the
affected body member. A Query without body members imposes no member counts.
Input or traversal failures remain execution errors and must not become a
successful Query-Body Validation report.

#### Complete Match Projection

The complete `CONSTRUCT` below brings these steps together for the sensor
example. It first selects the Point records that form `R`, then uses a `VALUES`
table to pair each body member with the types that fit it. The final filter
keeps records that claim the member, or matching free records if none claim it.

```sparql
CONSTRUCT { ?subject ?requirement ?result }
WHERE {
  # Existing traversal and of:Point matching produce R.
  VALUES ?subject { ex:equip }
  ?subject ex:Equipment.pointRefs ?result .
  ?result a sys:Entity .
  FILTER(isIRI(?result))
  FILTER EXISTS {
    ?result a ?resultType .
    VALUES ?resultType {
      ex:Point ex:Temperature ex:Pressure
      ex:Equipment.points.temperatureSensor ex:Equipment.points.pressureSensor
    }
  }
  # Each body member supplies its name and matching types.
  VALUES (?requirement ?matchingType) {
    (ex:Equipment.points.temperatureSensor ex:Temperature)
    (ex:Equipment.points.temperatureSensor ex:Equipment.points.temperatureSensor)
    (ex:Equipment.points.pressureSensor ex:Pressure)
    (ex:Equipment.points.pressureSensor ex:Equipment.points.pressureSensor)
  }
  # Count an explicit claim, or a free match if no result claims this member.
  FILTER (
    EXISTS { ?result sys:spec ?requirement }
    || (
      NOT EXISTS {
        ?result sys:spec ?claimedMember .
        VALUES ?claimedMember {
          ex:Equipment.points.temperatureSensor ex:Equipment.points.pressureSensor
        }
      }
      && EXISTS { ?result a ?matchingType }
      && NOT EXISTS {
        ?subject ex:Equipment.pointRefs ?claim .
        ?claim a sys:Entity ; sys:spec ?requirement .
        FILTER(isIRI(?claim))
        FILTER EXISTS {
          ?claim a ?claimType .
          VALUES ?claimType {
            ex:Point ex:Temperature ex:Pressure
            ex:Equipment.points.temperatureSensor ex:Equipment.points.pressureSensor
          }
        }
      }
    )
  )
}
```

The property and type lists show only the definitions needed for this example.
A translation must resolve the complete sets using the
[field-name](#resolving-a-field-name) and [type-membership](#type-membership)
rules. Claim tests must also include subtypes that claim a member through their
nominal base chain, as described in [Composing Body Validation](#composing-body-validation).

For a member with sugar constraints, replace `EXISTS { ?result a ?matchingType }`
with the full [sugar-matching predicate](#sugar-matching).

The `?claim` check searches all of `R` for a record that claims the current
member. Only claims within `R` take precedence over free matches; a record
outside the Query's results has no effect on that member's count.

### Result Validation

Result Validation checks the selected records against the Query's
`of` type. Use `sh:targetNode` to name each returned record to validate, and
`sh:node` to name the shape containing the constraints to apply. That constraint
shape has no targets of its own, so it does not independently select other
records for validation.

**Validate the selected records using the full graph.** The Query results
determine where Result Validation starts, not which triples the validator can access.
Keep all instance and schema triples available so reference constraints can
check other records as needed.

For the result `ex:t`:

```turtle
ex:SelectedResults a sh:NodeShape ;
  sh:targetNode ex:t ;
  sh:node ex:TargetResultConstraints .

ex:TargetResultConstraints a sh:NodeShape ;
  sh:property [
    sh:path sys:hasMarker ;
    sh:hasValue ex:Target.hit
  ] .
```

Target requires only the `hit` marker, represented here by `sh:hasValue`.

With the RDF above, Result Validation passes. Removing
`ex:t sys:hasMarker ex:Target.hit` leaves the query result unchanged but makes
Result Validation fail because `ex:t` no longer has Target's required `hit`
marker. Result Selection and Result Validation remain separate, as they are in Xeto.

#### Validating Sugar Results

A record selected by a sugar target may declare the sugar's nominal anchor
as its type. Result Validation must account for the field IRIs associated with
that declared type. For example, `candidate` in [Sugar Matching](#sugar-matching)
declares Base, so Result Validation against Sweet must recognize Base's field
IRIs as well as Sweet's.

Checking only `ex:Sweet.hot` would miss the candidate's
`sys:hasMarker ex:Base.hot` statement. Result Validation uses the following SHACL
shapes to check `candidate` against Sweet. The `hot` requirement accepts any
of the relevant marker IRIs:

```turtle
ex:SelectedSugarResults a sh:NodeShape ;
  sh:targetNode ex:candidate ;
  sh:node ex:SweetResultConstraints .

ex:SweetResultConstraints a sh:NodeShape ;
  sh:property [
    sh:path sys:hasMarker ;
    sh:hasValue ex:Base.required
  ] ;
  sh:property [
    sh:path sys:hasMarker ;
    sh:qualifiedValueShape [
      sh:in ( ex:Base.hot ex:Sweet.hot ex:MoreSweet.hot )
    ] ;
    sh:qualifiedMinCount 1
  ] .
```

With `required` and `hot` present as Markers, `candidate` passes both checks.
It is still selected if `required` is missing or `hot` contains a string, but
Result Validation fails: `required` must be present, and `hot` must be a Marker.

These shapes show only the marker checks. A complete validation shape must
check all required fields, value types, and invariants, including inherited
requirements, using the SHACL rules in the
[mapping chapters](doc.xeto.rdf::index).

Each field check must accept all IRIs resolved by the
[field-name rule](#resolving-a-field-name). For value-bearing fields, combine
the property IRIs in an alternative path. For Markers, use `sys:hasMarker`
and accept any of the applicable marker IRIs, as shown above.

#### Instance Data and Schema Vocabulary

A runtime record and a schema class can share the same IRI. When combining
instance and schema triples for validation, schema statements at that IRI must
not fill in missing fields on the record.

For example, these schema triples declare Base's required `hit` marker:

```turtle
ex:Base a rdfs:Class ;
  rdfs:subClassOf sys:Dict ;
  sys:hasMarker ex:Base.hit .
```

These instance triples describe a record also named `ex:Base`, but without `hit`:

```turtle
ex:Base a sys:Entity, ex:Base .
```

| How the validation data graph is built | Result of checking the required `hit` marker |
| --- | --- |
| **Bad:** merge all schema and instance triples. | Passes incorrectly: the schema's `sys:hasMarker` triple supplies the missing marker. |
| **Good:** keep the instance triples and only the schema's class and subclass declarations at the shared IRI. | Fails correctly: the record has no `hit` marker. |

Keep the full instance graph and supporting schema vocabulary available. At
each shared IRI, include only the schema's `rdf:type rdfs:Class` declaration
and `rdfs:subClassOf` statements in the validation data graph. These let the
validator identify the class targeted by a spec Ref and determine its
supertypes. Keep other schema statements about that IRI outside the data graph.
