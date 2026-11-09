# Object Slots

## Other Core Types

Most dictionary types need no special mapping. A type derived from `Dict` uses
the [class and slot rules](Mappings.md#specs-slots-and-instances) and the
[nested-object rules](#nested-object-slots). For example,
`Err` is a dictionary with `dis` and optional `errTrace` slots:

```xeto
Failure : Err

Envelope : Dict {
  failure: Failure?
}
```

```turtle
ex:Failure a sys:Class, rdfs:Class ;
  rdfs:subClassOf sys:Err ;
  rdfs:label "Failure"@en .

ex:Envelope a sh:NodeShape ;
  sh:targetClass ex:Envelope ;
  sh:property [
    sh:path ex:Envelope.failure ;
    sh:node ex:Failure ;
    sh:maxCount 1 ;
  ] .
```

The following core types do not have mappings in this version:

| Xeto type | Reason |
| --- | --- |
| `Grid` | A grid mapping must define rows, columns, cells, metadata, and ordering. |
| `Collection` | The abstract collection type does not identify a concrete RDF value representation. |
| `Func` | Function parameters, return values, and execution are outside the current data profiles. |
| `Interface`, `Funcs`, and `File` | API contracts, callable operations, and file handles are outside the current data profiles. |

An exporter reports that the affected spec is unsupported when one of these
types is used as an RDF value type. It does not treat the value as a string,
dictionary, or unconstrained object. `Entity`, `Err`, and other ordinary
dictionary types continue to use the `Dict` mapping.

This check follows the complete inheritance chain. For example, a
`SpecialGrid` derived from `Grid` is still a grid value, so a slot whose type is
`SpecialGrid` fails with the unsupported Grid diagnostic. Giving an unsupported
core type an application-specific name does not turn it into a dictionary or
an unconstrained object.

## Unconstrained Object Slots

### SHACL Validation

An `Obj` slot constrains presence and single-value cardinality but deliberately
does not constrain the RDF term's datatype, class, or node kind.

```xeto
Envelope : Dict {
  value: Obj
}
```

```turtle
ex:Envelope a sh:NodeShape ;
  sh:targetClass ex:Envelope ;
  sh:property [
    sh:path ex:Envelope.value ;
    sh:minCount 1 ;
    sh:maxCount 1 ;
  ] .
```

## Nested Object Slots

### SHACL Validation

A slot typed as another dictionary spec uses `sh:node` to apply that spec's
shape to the nested value.

```xeto
Address : Dict {
  city: Str
}

PostalAddress : Address

Person : Dict {
  address: Address
  postalAddress: PostalAddress?
}
```

```turtle
ex:Person a sh:NodeShape ;
  sh:targetClass ex:Person ;
  sh:property [
    sh:path ex:Person.address ;
    sh:node ex:Address ;
    sh:minCount 1 ;
    sh:maxCount 1 ;
  ] ;
  sh:property [
    sh:path ex:Person.postalAddress ;
    sh:node ex:PostalAddress ;
    sh:maxCount 1 ;
  ] .
```

Dictionary inheritance may pass through more than one spec. The exporter
follows the complete inheritance chain, but `sh:node` names the slot's declared
type; in this example, `PostalAddress`. If that type is declared by a dependency,
the same rule applies using the dependency library's RDF namespace. A loaded
`addresses::PostalAddress`, for example, becomes
`sh:node addresses:PostalAddress`.

The nested node may itself contain scalar, ref, marker, unit, list, choice, or
further nested values supported by this specification.

### Instance Data

A nested dictionary without its own Xeto id becomes an RDF blank node. It is
typed with its nested spec so both class-based and `sh:node` constraints can
recognize it.

```xeto
@person1: Person {
  address: Address {
    city: "Richmond"
  }
  postalAddress: PostalAddress {
    city: "Richmond"
  }
}
```

```turtle
ex:person1 a sys:Entity, ex:Person ;
  ex:Person.address [
    a ex:Address ;
    ex:Address.city "Richmond"^^xsd:string
  ] ;
  ex:Person.postalAddress [
    a ex:PostalAddress ;
    ex:Address.city "Richmond"^^xsd:string
  ] .
```

If the nested instance has its own Xeto id, that id becomes a named RDF
individual instead of a blank node.

```xeto
@person1: Person {
  address @address1: Address {
    city: "Richmond"
  }
}
```

```turtle
ex:person1 ex:Person.address ex:address1 .

ex:address1 a sys:Entity, ex:Address ;
  ex:Address.city "Richmond"^^xsd:string .
```

