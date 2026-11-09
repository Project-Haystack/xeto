# Deferred Mappings

The following forms do not have a conformance mapping in this version. An
exporter encountering one of them fails instead of silently emitting weaker
RDF or SHACL. The error identifies the unsupported construct and, when it is
available, the affected qualified spec or slot. This specification does not
define error codes or prescribe programming-language error types.

- Query-body checks in schema-only exports. The
  [Query-body validation](Queries.md#query-body-validation) operation produces separate
  SPARQL and SHACL for those checks.
- Named-result assignment through `queryNamed`, including its selection-order
  policy when several records match the same name or one record matches several
  names. Query-body validation does not assign one record to each name.
- Single-result selection through `query`, and display-name sorting and result
  limits through `queryAll`. The current mapping defines the complete set of
  selected identities. Choosing one result, sorting results, and limiting results
  require a separate host-executor policy; SPARQL `ORDER BY` and `LIMIT` alone
  do not establish compatibility with those Xeto operations.
- `List` item types for enums and choices, pending Xeto/Haystack instance
  fidelity fixes.
- `List` item types for units and refs, and nested `List` values, pending a
  complete instance and validation mapping.
- Reference lists stored in undeclared fields, pending a consistent distinction
  between runtime `MultiRef` values and `List<of:Ref>` values during export.
  Declared `MultiRef` slots remain supported.
- Direct `Scalar` slots, pending a SHACL mapping for the abstract family of
  special and custom representations. Concrete custom scalars use the mapping
  in [Custom Scalar Datatypes](Scalars.md#custom-scalar-datatypes).
- `Grid` and the abstract `Collection` type, pending a complete collection and
  table mapping.
- `Func`, `Interface`, and `Funcs`, whose API and execution semantics are
  outside the current data profiles.
- `sys::This`, `Ref<of:This>`, and `MultiRef<of:This>` slots, pending a
  parent-relative RDF and SHACL mapping.
- Runtime component and function-block constructs from `sys.comp`.

Translating a Xeto `Query` into executable SPARQL and companion SHACL is
specified separately from the three RDF export profiles. See
[Query Execution and Result Validation](Queries.md#query-execution-and-result-validation).
