<!--
title:      Query Result Selection
-->

After Input Validation passes, the Query executor runs a SPARQL `SELECT`
over the instance graph. The SELECT binds the starting record's IRI to
`?subject` and uses the Query path to reach `?result`. Each reached record
must match the Query's `of` type to be returned. `SELECT DISTINCT` returns
each matching record's IRI once.

The [worked SELECT](QueryExecution.md#result-selection) shows these parts
together.

| Part | Translation rule | What supplies its contents? |
| --- | --- | --- |
| Starting-record binding | Bind `?subject` to the supplied record IRI, using an RDF-term binding or a safely serialized `VALUES` clause. | The execution request |
| [Traversal](QueryTraversal.md) | Relate `?subject` to `?result`, or bind them to the same identity for `id`. | The Query's path, direction, repetition, and resolved field properties |
| Record restriction | Require `?result a xetoRdf:Record` and `isIRI(?result)`. | Fixed structure; the record class uses the RDF mapping namespace |
| [Result matching](QueryResultMatching.md) | Require the reached record to fit the Query's `of` type. | The target type and resolved schemas |
| Unique-result projection | Return `SELECT DISTINCT ?result`, with no ordering guarantee. | Fixed structure |

Fill in the bracketed parts of the SELECT template using the rules in the table.

```text
SELECT DISTINCT ?result WHERE {
  [starting-record binding]
  [traversal]
  ?result a xetoRdf:Record .
  FILTER(isIRI(?result))
  FILTER([result-matching predicate])
}
```
