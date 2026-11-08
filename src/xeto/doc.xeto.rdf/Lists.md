# Lists

## SHACL Validation

A Xeto `List` is one ordered collection, not several independent values for the
same slot. RDF represents that collection as a linked list: each node stores
one item in `rdf:first` and points to the remainder of the list through
`rdf:rest`. The exporter checks each `rdf:first` value against the Xeto `of`
type. It counts the linked-list nodes separately so repeated values still count
as separate list items when enforcing `minSize` and `maxSize`.

```xeto
Batch : Dict {
  names: List <of:Str, minSize:2, maxSize:3>
}
```

```turtle
ex:Batch a sh:NodeShape ;
  sh:targetClass ex:Batch ;
  sh:property [
    sh:path ex:Batch.names ;
    sh:minCount 1 ;
    sh:maxCount 1 ;
    sh:node [
      sh:property [
        sh:path ( [ sh:zeroOrMorePath rdf:rest ] rdf:first ) ;
        sh:datatype xsd:string ;
      ] ;
      sh:property [
        sh:path [ sh:zeroOrMorePath rdf:rest ] ;
        sh:minCount 3 ;
        sh:maxCount 4 ;
      ]
    ] ;
  ] .
```

An RDF list containing `n` items has `n + 1` nodes because the chain ends at
the additional `rdf:nil` node. Xeto's `minSize` and `maxSize` item counts are
therefore increased by one when converted to these SHACL node counts. In the
example above, `minSize:2` becomes `sh:minCount 3`, and `maxSize:3` becomes
`sh:maxCount 4`.

`nonEmpty` requires at least one list item. Its SHACL node count is two: one
item node plus the terminating `rdf:nil` node.

```xeto
Batch : Dict {
  tags: List <of:Str, nonEmpty>
}
```

```turtle
ex:Batch a sh:NodeShape ;
  sh:targetClass ex:Batch ;
  sh:property [
    sh:path ex:Batch.tags ;
    sh:minCount 1 ;
    sh:maxCount 1 ;
    sh:node [
      sh:property [
        sh:path ( [ sh:zeroOrMorePath rdf:rest ] rdf:first ) ;
        sh:datatype xsd:string ;
      ] ;
      sh:property [
        sh:path [ sh:zeroOrMorePath rdf:rest ] ;
        sh:minCount 2 ;
      ]
    ] ;
  ] .
```

When `of` names a dictionary spec, every list item must be an instance of that
spec, so the exporter uses `sh:class`. Scalar item types instead use
`sh:datatype`.

Lists of enum values, Choice selections, units, references, or other lists are
not supported in this version. Exporting one of these list forms is an error;
see [Deferred Mappings](Deferred.md#deferred-mappings).

## Instance Data

A Xeto list becomes an RDF collection, preserving item order.

```xeto
@batch1: Batch {
  names: { "alpha", "beta" }
}
```

```turtle
ex:batch1 a sys:Entity, ex:Batch ;
  ex:Batch.names ("alpha"^^xsd:string "beta"^^xsd:string) .
```

The parenthesized Turtle syntax is shorthand for an `rdf:first`/`rdf:rest`
chain. The same collection can be written out as:

```turtle
ex:batch1 ex:Batch.names _:names1 .

_:names1 rdf:first "alpha"^^xsd:string ;
  rdf:rest _:names2 .

_:names2 rdf:first "beta"^^xsd:string ;
  rdf:rest rdf:nil .
```

The final `rdf:rest rdf:nil` is implicit in the parenthesized form. A malformed
chain that never reaches `rdf:nil` is not a well-formed RDF list.

