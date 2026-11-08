# Overview

Xeto maps specs and instances to RDF for interoperability with RDF-based systems.
The mapping uses RDFS for type relationships, SHACL for validation, and SPARQL
for executable Queries.

This chapter introduces the main transformations. The detailed specification
lives in [doc.xeto.rdf::Overview].

## Specs Become Classes and Properties

Consider a Person with a required name, and one instance:

```xeto
Person: Dict {
  name: Str
}
@ada: Person { name: "Ada" }
```

The spec becomes an RDF class, and the slot becomes an RDF property:

```turtle
ex:Person a sys:Class, rdfs:Class ;
  rdfs:subClassOf sys:Dict ;
  rdfs:label "Person"@en .

ex:Person.name a rdf:Property ;
  rdfs:label "name"@en .
```

`ex:Person.name` identifies the slot on Person. Another spec's `name` slot can
have a different property, so the two field definitions remain distinct.
Xeto inheritance becomes `rdfs:subClassOf` relationships.

These examples use Turtle, a text format for RDF. `ex:` abbreviates the example
library's versioned namespace; `sys:` abbreviates the resolved sys library's
namespace. The `rdf:`, `rdfs:`, `sh:`, and `xsd:` prefixes name standard
vocabularies. The examples show the statements relevant to each transformation;
a full export includes additional metadata.

See [Specs, Slots, and Instances](doc.xeto.rdf::Mappings#specs-slots-and-instances)
and [Namespaces and IRIs](doc.xeto.rdf::Foundations#namespaces-and-iris).

## Instances Become Triples

The same `@ada` instance becomes:

```turtle
ex:ada a sys:Entity, ex:Person ;
  ex:Person.name "Ada" ;
  sys:spec ex:Person ;
  sys:hasRefField sys:spec .
```

Each RDF statement has a subject, a property, and a value. For example,
`ex:ada ex:Person.name "Ada"` says that Ada's name is the string "Ada".
The Turtle shorthand `a` means `rdf:type`: Ada is a Person and an identified
record. The instance ID becomes the subject IRI, `ex:ada`.

The declaration `@ada: Person` implicitly supplies a `spec` field. If Person
belongs to `example`, the record looks like this in Trio:

```trio
id: @example::ada
spec: @example::Person
name: "Ada"
```

The `@example::Person` value is a Ref. `sys:spec ex:Person` preserves that
value, and `sys:hasRefField sys:spec` records that the `spec` field contains
a Ref. Type membership and the actual Ref are kept separate so a Query
can follow the stored reference without also following inferred types.
See [The spec Field](doc.xeto.rdf::Mappings#the-spec-field).

## Constraints Become SHACL Shapes

`name: Str` requires one string value. Its SHACL mapping is:

```turtle
ex:Person a sh:NodeShape ;
  sh:targetClass ex:Person ;
  sh:property [
    sh:path ex:Person.name ;
    sh:datatype xsd:string ;
    sh:minCount 1 ;
    sh:maxCount 1
  ] .
```

The shape checks Person records. Ada passes because the record has one string
name. Removing the name or replacing it with an integer fails validation.
An RDF type assertion alone does not check required fields; the SHACL shape
performs those checks.

The vocabulary, instance data, and shapes have different roles, even when an
export puts all three in one Turtle file. See
[RDF Graphs](doc.xeto.rdf::Foundations#rdf-graphs) for how consumers use them,
and [Mapping Rules](doc.xeto.rdf::index) for each Xeto construct's constraints.

## Queries Become SPARQL

Using Person and Ada above, a directory can refer to Ada and declare a Query
that follows the reference:

```xeto
Directory: Dict {
  personRef: Ref<of:Person>
  people: Query<of:Person, via:"personRef">
}
@directory1: Directory { personRef: @ada }
```

Starting `Directory.people` at `directory1` returns `ada`. The stored link is
`ex:directory1 ex:Directory.personRef ex:ada`. The corresponding SELECT is:

```sparql
SELECT DISTINCT ?result WHERE {
  VALUES ?subject { ex:directory1 }
  ?subject ex:Directory.personRef ?result .
  ?result a sys:Entity .
  FILTER(isIRI(?result))
  FILTER EXISTS {
    ?result a ?type . VALUES ?type { ex:Person }
  }
}
```

The SELECT returns `ex:ada`. The displayed property and type lists cover this
example; compilation resolves those lists from the selected namespace.

Execution checks the starting record and followed references before running
selection. When result validation is requested, SHACL then checks the returned
records. A record that fails result validation remains in the results, with its
validation failure reported alongside it.

See [Query Execution and Result Validation](doc.xeto.rdf::Queries) for the
complete workflow, including input shapes, inverse paths, repeated traversal,
and Query-body requirements.

## Detailed Specification

- [Overview](doc.xeto.rdf::Overview): supported profiles, conformance, and scope.
- [Foundations](doc.xeto.rdf::Foundations): resolved libraries, graphs, and identifiers.
- [Metadata](doc.xeto.rdf::Metadata): metadata and links to external vocabularies.
- [Mappings](doc.xeto.rdf::index): rules and examples for individual Xeto constructs,
  including units and quantities mapped to QUDT.
- [Queries](doc.xeto.rdf::Queries): executable Query translation and validation.
- [Deferred Mappings](doc.xeto.rdf::Deferred): forms without a supported mapping.
