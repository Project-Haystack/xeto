# Query Result Matching

Result matching decides which records reached by traversal are returned,
using the Query's `of` type. For an ordinary named type such as [`Target`](QueryExecution.md#worked-example), a
reached record matches when its declared type is Target or a subtype of Target.
Required fields are checked separately during [Result Validation](QueryResultValidation.md#query-result-validation).

Sugar types also allow a reached record to match based on the record's field
values, even when the record's declared type is not a subtype of the sugar type.
Non-sugar unions and intersections use type membership. The rules below define
how each target forms the SELECT's result-matching predicate.

## Type Membership

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
type. In the [worked example](QueryExecution.md#worked-example),
`type_match(Target)` is the complete result predicate.

## Sugar Matching

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
using the [field-name rule](QueryTraversal.md#resolving-a-field-name). Then apply the corresponding
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

### Result Selection and Result Validation

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

### Computed Sugar Subtyping

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

### Invariant Values

An invariant, unlike a default, adds a value-equality test to sugar's structural
alternative. For example:

```xeto
Named: Base <sugar> {
  label: Str <invariant> "wanted"
}
```

For records declared as `Base`, the structural alternative for `of:Named`
requires `label` to equal `"wanted"`. If [`candidate`](#sugar-matching) is queried
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

## Union Targets

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

## Intersection Targets

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

## Records Declared with Compound Types

[Union Targets](#union-targets) and [Intersection Targets](#intersection-targets)
use a compound as the Query's target type. A record's declared type can also
be a compound. In that case, Result Selection depends on what the declared
type guarantees:

- `Both: Base & Other` guarantees membership in both Base and Other. A record
  declared as `Both` therefore matches a Query targeting either type. It also
  matches `Either: Sweet | Other` through its Other member.
- `Either: Sweet | Other` guarantees membership in one alternative, but does
  not say which one. A record declared as `Either` cannot match a Query targeting
  Base by type membership: Sweet fits Base, but Other does not.

An intersection's declared type can therefore fit a target through one member;
a union's declared type fits a target only if every alternative fits it.

For `of:Base`, the class list in [Type Membership](#type-membership) includes
Both but excludes Either. An abbreviated replacement for the `VALUES` list is:

```sparql
VALUES ?type { ex:Base ex:Both }
```

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

## Choice Targets

Choice targets use [Type Membership](#type-membership). For example:

```xeto
Flavor: Choice
Vanilla: Flavor { vanilla }
```

For `of:Flavor`, replace the matching-class list in the Type Membership
pattern with:

```sparql
VALUES ?type { ex:Flavor ex:Vanilla }
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

## Scalar Targets

> **Translation error:** Scalar Query targets are not supported by this mapping.
> The translator must reject a Query with a scalar `of` type; see
> [Deferred Mappings](Deferred.md#deferred-mappings).

Scalar targets do not match ordinary records. However, a malformed runtime
record can declare a scalar type:

```trio
id: @invalid
spec: @sys::Str
```

If traversal reaches this record, native Xeto selects it for `of:Str` through
[Type Membership](#type-membership). The record still fails validation:
declaring Str does not make a record a string.

Returning an empty result instead of a translation error would disagree with
native Query execution for records such as `invalid`.
