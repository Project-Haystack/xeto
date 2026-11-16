# Query Result Validation

Result Selection has already established that each returned record fits the
Query's `of` type. Optional Result Validation checks the `of` type's field
requirements on those records. Use `sh:targetNode` to name each returned record
to validate, and `sh:node` to name the shape containing the constraints to apply. That constraint
shape has no targets of its own, so it does not independently select other
records for validation.

**Keep the full instance graph available during validation.** The Query results
determine where Result Validation starts, not which triples the validator can access.
Reference constraints may need other records. Include supporting schema
vocabulary using the [shared-IRI rules](#instance-data-and-schema-vocabulary),
so schema metadata cannot supply missing record fields.

For the result [`ex:t`](Queries.md#worked-example):

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

Result Validation does not change Result Selection or which Query-body member
a record claims. Matching records may use different field IRIs, as shown in
[Validating Sugar Results](#validating-sugar-results).

## Validating Sugar Results

A record selected by a sugar target may declare the sugar's nominal anchor
as its type. Result Validation must account for the field IRIs associated with
that declared type. For example, `candidate` in [Sugar Matching](QueryResultMatching.md#sugar-matching)
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

## Instance Data and Schema Vocabulary

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
