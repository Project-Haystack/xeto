# Metadata and External Vocabularies

The RDF mapping does not require OWL reasoning. An exporter may describe a
library resource as an `owl:Ontology` and may use `owl:imports` as packaging
metadata, but these statements are not required for vocabulary, SHACL, or
instance-profile conformance.

Each value-bearing Xeto slot becomes an RDF property that connects an instance
to a value. For example, `siteRef` connects an equipment instance to its site:

```turtle
lib:equip1 lib:Equip.siteRef lib:site1 .
```

The vocabulary export also describes that property so RDF tools can recognize
and display it:

```turtle
lib:Equip.siteRef a rdf:Property ;
  rdfs:label "siteRef"@en .
```

The first statement contains instance data. The second identifies
`lib:Equip.siteRef` as an RDF property and gives it the English label
`siteRef`; the `@en` suffix marks the label as English. Every exported slot
property includes this description. Its SHACL constraints are defined
separately.

Xeto names and documentation map like this:

```xeto
Person : Dict <doc:"A person">
```

```turtle
lib:Person rdfs:label "Person"@en ;
  rdfs:comment "A person"@en .
```

In an RDF statement, the predicate names the relationship between the subject
and object. In this example, `rdfs:label` and `rdfs:comment` are the predicates.
They relate `lib:Person` to its display label and documentation text.

## Metadata Without an RDF Mapping

Xeto libraries can add their own metadata to specs and slots. For example, an
application can define an `icon` tag and use it to select an icon in its user
interface:

```xeto
+Spec {
  icon: Str?
}

Person : Dict <icon:"user"> {
  dis: Str <icon:"label">
}
```

This specification does not assign an RDF meaning to `icon`, so the RDF is the
same as if the `icon` metadata were not present:

```turtle
lib:Person a sys:Class, rdfs:Class ;
  rdfs:subClassOf sys:Dict ;
  rdfs:label "Person"@en .

lib:Person.dis a rdf:Property ;
  rdfs:label "dis"@en .
```

The `+Spec` declaration does not create an RDF property or a SHACL shape for
`icon`, and the two uses of `icon` do not create RDF statements.

An exporter omits metadata for which this specification defines no mapping. It
still exports the containing spec or slot. This rule applies to
application-defined metadata and to standard Xeto metadata such as `summary` or
`readonly` when no mapping is defined here.

Metadata used by a mapping in this specification is handled strictly. For
example, `minVal`, `of`, `via`, `quantity`, and `unit` affect the generated
SHACL graph. An exporter reports an error when mapped metadata has an invalid
type or value; it does not ignore that error and emit a weaker constraint.

References to external vocabularies use their canonical published IRIs. Xeto
vocabulary, SHACL, and instance exports do not copy declarations or
classification facts owned by those vocabularies. A validation environment
loads the required external vocabulary graphs separately.

For example, a temperature unit shape may require a QUDT unit with the
temperature quantity kind:

```turtle
lib:Sensor a sh:NodeShape ;
  sh:property [
    sh:path lib:Sensor.unit ;
    sh:class qudt:Unit ;
    sh:node [
      sh:property [
        sh:path (
          qudt:hasQuantityKind
          [ sh:zeroOrMorePath skos:broader ]
        ) ;
        sh:hasValue quantitykind:Temperature ;
      ] ;
    ] ;
  ] .
```

During validation, the validator also needs the relevant facts from QUDT, such
as the unit's type and quantity-kind relationship:

```turtle
unit:DEG_F a qudt:Unit ;
  qudt:hasQuantityKind quantitykind:Temperature .
```

The path also accepts more specific QUDT quantity kinds. This rule and its
required QUDT data are explained under
[Units and Quantities](UnitsAndQuantities.md#quantity-kind-hierarchy).

The validator loads these facts from QUDT separately. It loads both the unit
graph and, when currency values are present, the currency graph. An instance
export may refer to `unit:DEG_F` or `currency:USD`, but it does not copy
QUDT-owned facts into the instance graph. The exact correspondence is defined
by the canonical machine-readable
[unit mapping](https://github.com/Project-Haystack/xeto/blob/master/src/xeto/sys.rdf/qudt-units.props)
and
[quantity-kind mapping](https://github.com/Project-Haystack/xeto/blob/master/src/xeto/sys.rdf/qudt-quantities.props)
packaged by Xeto's `sys.rdf` library.

