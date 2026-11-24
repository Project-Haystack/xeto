<!--
title:      Query Input Validation
-->

Input Validation checks whether the Query can run. The Query executor
generates a SHACL input shape for the starting record and Query path to
check the RDF representation of the path. Apply the shape to the instance
graph to check that the starting record exists and apply the reference checks
required by the path. Run the SPARQL `SELECT` only if Input Validation passes.

The Query executor runs this input shape against the instance graph for
[`Holder.direct` starting at `a`](QueryExecution.md#worked-example). The example
abbreviates the generated SHACL. The opening `[]` is a blank node for the shape.
`sh:targetNode ex:a` identifies the record to check.

```turtle
[] a sh:NodeShape ;
  sh:targetNode ex:a ;

  # Require ex:a to be an IRI identifying an exported record.
  sh:nodeKind sh:IRI ;
  sh:property [ sh:path rdf:type ; sh:hasValue xetoRdf:Record ] ;

  # Require each next value to be an IRI.
  sh:property [ sh:path ex:Holder.next ; sh:nodeKind sh:IRI ] ;

  # Reject a Marker stored as next through xetoRdf:hasMarker.
  sh:property [
    sh:path xetoRdf:hasMarker ;
    sh:not [ sh:in ( ex:Holder.next ) ]
  ] .
```

Any failed check is an execution error.

For `via:"next+"`, put the `next` field checks from the shape above inside
`sh:node` in this template.

```text
sh:property [
  sh:path [ sh:zeroOrMorePath ex:Holder.next ] ;
  sh:node [ ... ]
]
```

Zero hops checks `next` on the starting record, where the Query reads the first
link. The field checks apply only to the start and the records reached by
following `next`.

| Input | Required behavior |
| --- | --- |
| Starting record `a` is missing | Report an execution error. |
| `a.next` is a Ref to an existing `t: Target` | Pass Input Validation. |
| `a.next` is a Ref to a missing record | Pass Input Validation. |
| `a.next` stores a literal or a nested Dict | Report an error when trying to follow it. |
| For `via:"next+"`, `a.next` is a Ref to `b`, and `b.next` contains a string | Report an execution error before running the SELECT. |
| A Ref reaches `t: Target`, but `t` lacks `hit` | Pass Input Validation. |

When SHACL checks `next` on a record, apply the reference checks to every RDF
property that [maps to `next`](QueryTraversal.md#resolving-a-field-name), such as
`ex:Holder.next` and `ex:Bridge.next`. See [Query Traversal](QueryTraversal.md)
for the corresponding SPARQL paths.

Suppose `next: Unit` holds the pressure unit pascal. The record is valid.
What happens when a Query follows `via:"next"`?

- **Xeto** raises an error. It cannot follow a unit as a Ref.
- **SHACL** accepts the link. Pascal is written as `unit:PA`, which passes the IRI check.

A Query can start at a record of any type. If a followed field contains several
Refs, native Xeto tries each one, so Input Validation accepts them without
enforcing the field's declared count.

# Inverse Paths

A one-hop inverse Query first checks that the starting record exists.
It then looks for records that point back to it:

- **Xeto** matches a Ref to the starting record.
- **SPARQL** matches a link to the starting record's IRI. A scalar stored
  with that same IRI also matches.

A **repeated inverse** path checks the followed field on every exported record.
For example, add this runtime record to either example under
[Inverse Result Selection](QueryTraversal.md#inverse-result-selection).

```trio
id: @example.inverse::unrelated
equipRef: "not a Ref"
```

A repeated inverse Query looks for every record with an `equipRef` chain back
to `equip1`. Any record with `equipRef` could start such a chain, so Input
Validation checks that every `equipRef` value is an IRI and rejects Marker fields before the SELECT runs.
The example record has no path to `equip1`. The string fails the check and
causes an execution error. Use `sh:targetClass xetoRdf:Record` to run the check
on all exported records.

# Identity Paths

For [identity paths](QueryTraversal.md#identity-traversal), check only that
the starting record exists. This applies to `id`, `id+`, and their inverses.

# Spec-Reference Paths

Apply the reference checks to `sys:spec` when the Query follows `spec`. Use
the [inverse checks](#inverse-paths) for inverse `spec` paths.

In the [spec-reference example](QueryTraversal.md#spec-reference-traversal),
`transitive` starts at `reading` and follows `sys:spec`.

```turtle
[] a sh:NodeShape ;
  sh:targetNode ex:reading ;

  # Require ex:reading to be an IRI identifying an exported record.
  sh:nodeKind sh:IRI ;
  sh:property [ sh:path rdf:type ; sh:hasValue xetoRdf:Record ] ;

  # Check spec on the starting record and every record reached through sys:spec.
  sh:property [
    sh:path [ sh:zeroOrMorePath sys:spec ] ;
    sh:node [
      # Require each spec value to be an IRI.
      sh:property [ sh:path sys:spec ; sh:nodeKind sh:IRI ] ;

      # Reject spec stored as a Marker.
      sh:property [ sh:path xetoRdf:hasMarker ; sh:not [ sh:in ( sys:spec ) ] ]
    ]
  ] .
```
