<!--
title:      Query-Body Validation
-->

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

# Composing Body Validation

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
existing [result-matching rules](QueryResultMatching.md), including sugar constraints.

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

# Representing Matches for SHACL

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
existing Result Selection with the member-matching and claim rules in
[Composing Body Validation](#composing-body-validation).

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

# Complete Match Projection

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
[field-name](QueryTraversal.md#resolving-a-field-name) and [type-membership](QueryResultMatching.md#type-membership)
rules. Claim tests must also include subtypes that claim a member through their
nominal base chain, as described in [Composing Body Validation](#composing-body-validation).

For a member with sugar constraints, replace `EXISTS { ?result a ?matchingType }`
with the full [sugar-matching predicate](QueryResultMatching.md#sugar-matching).

The `?claim` check searches all of `R` for a record that claims the current
member. Only claims within `R` take precedence over free matches; a record
outside the Query's results has no effect on that member's count.
