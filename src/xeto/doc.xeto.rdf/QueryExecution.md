<!--
title:      Query Execution
-->

Executing a Xeto Query over exported RDF takes several steps. First, SHACL
checks whether the Query can run. If those checks pass, a SPARQL `SELECT` finds
the matching records. If the caller requests Result Validation, SHACL checks
the selected records.

The **Query executor** is the part of an implementation that takes an
[execution request](#execution-request), coordinates these stages, and reports
the outcome. This specification defines what the executor receives, runs, and
reports. Each implementation chooses how callers invoke the executor.

Query results are undefined if the input records fail validation against their
declared specs. To check the records first, run their SHACL shapes.
For a slot declared `siteRef: Ref`:

- `siteRef: @hq` supplies a Ref to the record named `hq`.
- `siteRef: "hq"` fails validation because `"hq"` is a string.

| Stage | What it does |
| --- | --- |
| [Input Validation](#input-validation) (SHACL) | Check the starting record and the path. Stop with an error if either check fails. |
| [Result Selection](#result-selection) (SPARQL) | Follow the path, returning distinct IRIs for records matching `of`. Order is unspecified. |
| [Result Validation](#result-validation) (SHACL, optional) | Check each returned record against `of`. Keep the record and report any failures. |

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

[Query Slot Mapping](QuerySlots.md#shacl-validation) describes schema paths.

Use the RDF export's [resolved library versions](Foundations.md#concrete-library-versions).
Report a [translation error](Overview.md#conformance) if the Query's behavior
cannot be preserved.

# Execution Request

A Query begins with a request to a query executor. This **execution request**
supplies:

| Input | Meaning |
| --- | --- |
| Query definition | The specific Query slot to execute, such as `ex::Holder.direct`. |
| Starting record | The RDF IRI of the record from which traversal starts. |
| Instance graph | The records to search, exported using the [instance profile](Overview.md). |
| [Resolved namespace](Foundations.md#namespace) | The library definitions and concrete versions used for the instance export, including active mixins. |
| Validate results | Boolean: run Result Validation when `true`; skip it when `false`. |
| Validation vocabulary | When requesting Result Validation, supply the schema and external vocabulary the result shape uses, including QUDT definitions for unit-bearing values. |

All stages must use the same Query, starting record, instance snapshot, and
resolved namespace. Input Validation and Result Selection use only the
instance graph. Result Validation also uses
[supporting vocabulary](QueryResultValidation.md#instance-data-and-schema-vocabulary).

# Worked Example

This example shows how the RDF export feeds each stage of Query execution.
Run `Holder.direct` from `a` with Result Validation requested.

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

Export the records to RDF using the instance profile. Run the SELECT against
the resulting RDF. Result Selection must use only exported instance triples.
Do not add schema triples or triples derived from schema relationships.

In the Turtle excerpt from the instance export, `ex:` is the example library's
concrete versioned namespace and `sys:` is the resolved sys library namespace.
Standard prefixes follow [Namespaces and IRIs](Foundations.md#namespaces-and-iris).

```turtle
ex:a a xetoRdf:Record, ex:Holder ;
  ex:Holder.next ex:t .
ex:t a xetoRdf:Record, ex:Target ;
  xetoRdf:hasMarker ex:Target.hit .
```

```text
Prepare
    Translate Holder.direct in the resolved namespace to build the SELECT
    and SHACL constraints.
    Export a and t to build the RDF instance graph.

Run
    Input Validation (SHACL) checks the instance graph.
        │ If input checks pass, run the SELECT.
        ▼
    Result Selection (SPARQL) reads the instance graph and returns ex:t.
        │ If Result Validation was requested, use ex:t as the SHACL target.
        ▼
    Result Validation (SHACL) checks ex:t using the full instance graph
    and supporting vocabulary, then reports failures.
```

## Input Validation

SHACL checks that `ex:a` exists and that each stored `next` value is an IRI.
See [Query Input Validation](QueryInputValidation.md) for the shape and path-specific checks.

## Result Selection

The SELECT abbreviates the property and class lists.

```sparql
SELECT DISTINCT ?result WHERE {             # Unique-result projection
  VALUES ?subject { ex:a }                  # Starting-record binding
  ?subject ex:Holder.next ?result .         # Traversal
  ?result a xetoRdf:Record .                    # Record restriction
  FILTER(isIRI(?result))
  FILTER EXISTS {                          # Result matching
    ?result a ?resultType . VALUES ?resultType { ex:Target }
  }
}
```

The SELECT follows `ex:Holder.next` from `ex:a` and returns `ex:t` because
`ex:t` has type `ex:Target`. See [Query Result Selection](QueryResultSelection.md)
for the SELECT template and clause rules.

## Result Validation

SHACL targets the returned `ex:t` and checks for the required
`xetoRdf:hasMarker ex:Target.hit` statement. If `hit` is missing, `ex:t` stays
in the results and the report flags the missing marker. See
[Query Result Validation](QueryResultValidation.md) for the shape and graph rules.

# Query-Body Validation

[Query-Body Validation](QueryBodyValidation.md) checks requirements such as
exactly one match for `temperatureSensor: Temperature`. The workflow takes
the Query slot, starting record IRI, instance graph, and resolved namespace.

After Input Validation passes, SPARQL `CONSTRUCT` writes each member's matches
as temporary RDF links. SHACL counts the links to report missing required
matches or ambiguous members. Records with invalid fields can still count.
