# Query Execution

This chapter defines how to execute Xeto Queries over exported RDF using
SPARQL, with SHACL checks before and after Result Selection. SPARQL follows the
Query's path and selects records matching its `of` type. SHACL checks whether execution
can proceed and, when requested, validates the returned records. Execution
errors and validation failures are reported separately.

Query execution has three stages: [Input Validation](QueryTraversal.md#input-validation), Result Selection, and
optional [Result Validation](QueryResultValidation.md#query-result-validation):

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

**Query-body counts are checked separately from result fields.**
[Query-Body Validation](QueryBodyValidation.md#query-body-validation) runs Input Validation and
Result Selection, then reports missing or ambiguous body members.
`Validate results` does not request these count checks, and records with
invalid fields still count.

**All stages must use the same Query, starting record, instance snapshot, and
resolved namespace; but different graph inputs.**
Input Validation and Result Selection read instance-export triples. Query-Body
Validation adds match information in a separate temporary graph. Result
Validation also uses supporting schema vocabulary; see
[Instance Data and Schema Vocabulary](QueryResultValidation.md#instance-data-and-schema-vocabulary).

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
| [Input Validation](QueryTraversal.md#input-validation) | SHACL checks that `ex:a` exists and that `next` contains references, using `sys:hasRefField` and the reference values. |
| [Result Selection](#result-selection) | SPARQL follows `ex:Holder.next` and returns `ex:t` because `ex:t` has type `ex:Target`. |
| [Result Validation](QueryResultValidation.md#query-result-validation) | SHACL checks that `ex:t` has the required `sys:hasMarker ex:Target.hit` statement. |

## Result Selection

For `Holder.direct` in the [worked example](QueryExecution.md#worked-example), Result
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
| [Traversal](QueryTraversal.md#traversal) | Relate `?subject` to `?result`, or bind them to the same identity for `id`. | The Query's path, direction, repetition, and resolved field properties |
| Record restriction | Require `?result a sys:Entity` and `isIRI(?result)`. | Fixed structure; the sys namespace uses its resolved version |
| [Result matching](QueryResultMatching.md#query-result-matching) | Require the reached record to fit the Query's `of` type. | The target type and resolved schemas |
| Unique-result projection | Return `SELECT DISTINCT ?result`, with no ordering guarantee. | Fixed structure |

The pieces compose as follows. This is a translation template, not executable
SPARQL; the bracketed parts stand for the constructs defined in
[Traversal](QueryTraversal.md#traversal) and
[Result Matching](QueryResultMatching.md#query-result-matching).

```text
SELECT DISTINCT ?result WHERE {
  [starting-record binding]
  [traversal]
  ?result a sys:Entity .
  FILTER(isIRI(?result))
  FILTER([result-matching predicate])
}
```

## Validation

[Input Validation](QueryTraversal.md#input-validation) asks whether execution can proceed. [Query-Body Validation](QueryBodyValidation.md#query-body-validation)
checks how many selected records match each required or optional member.
[Result Validation](QueryResultValidation.md#query-result-validation) asks whether the selected records conform to `of`.
These checks have different targets and must remain separate; a matching
record's invalid contents are not an input error unless traversal itself
encounters an invalid reference value.
