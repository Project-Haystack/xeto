# SHACL Validation

A Xeto `List` is one ordered collection, not several independent values for the
same slot. RDF represents that collection as a linked list: each node stores
one item in `rdf:first` and points to the remainder of the list through
`rdf:rest`. SHACL checks each `rdf:first` value against the Xeto `of` type.
The size constraints count list nodes, so repeated values still count as
separate items.

```xeto
Batch : Dict {
  names: List <of:Str, minSize:2, maxSize:3>
}
```

The `Batch` record shape applies `ex:Batch-fields` through `sh:node`. See
[Reusable Field Shapes](Mappings.md#reusable-field-shapes).

```turtle
ex:Batch-fields a sh:NodeShape ;
  sh:property [
    sh:path ex:Batch.names ;
    # The required names slot holds one list.
    sh:minCount 1 ;
    sh:maxCount 1 ;
    sh:node [
      # Check the item at rdf:first on each list cell.
      sh:property [
        sh:path ( [ sh:zeroOrMorePath rdf:rest ] rdf:first ) ;
        sh:datatype xsd:string ;
      ] ;
      sh:property [
        sh:path [ sh:zeroOrMorePath rdf:rest ] ;
        # Require the chain to reach rdf:nil.
        sh:hasValue rdf:nil ;
        # Each item cell has one item and one link. rdf:nil has neither.
        sh:or (
          [ sh:in ( rdf:nil ) ;
            sh:property [ sh:path rdf:first ; sh:maxCount 0 ], [ sh:path rdf:rest ; sh:maxCount 0 ] ]
          [ sh:not [ sh:in ( rdf:nil ) ] ;
            sh:property [ sh:path rdf:first ; sh:minCount 1 ; sh:maxCount 1 ],
                        [ sh:path rdf:rest ; sh:minCount 1 ; sh:maxCount 1 ] ]
        ) ;
        # Count the two or three item cells plus rdf:nil.
        sh:minCount 3 ;
        sh:maxCount 4 ;
      ]
    ] ;
  ] .
```

Every list slot requires the `rdf:rest*` path from its value to reach `rdf:nil`.
The empty list `()` reaches itself in zero steps. A Ref to an ordinary address
record fails this check even when the record exists.

Each list cell has exactly one `rdf:first` and one `rdf:rest`. The terminator
`rdf:nil` has neither. A cell written as `[ rdf:rest rdf:nil ]` fails validation
because the cell has no item.

An RDF list containing `n` items has `n + 1` nodes because the chain ends at
the additional `rdf:nil` node. Xeto's `minSize` and `maxSize` item counts are
therefore increased by one when converted to these SHACL node counts. In the
example above, `minSize:2` becomes `sh:minCount 3`, and `maxSize:3` becomes
`sh:maxCount 4`.

`nonEmpty` requires at least one list item.

```xeto
Batch : Dict {
  tags: List <of:Str, nonEmpty>
}
```

The `tags` shape uses the same item and chain checks as `names`. The size
check on the `rdf:rest*` path has a minimum of two nodes.

```turtle
# Count one item cell and the terminating rdf:nil node.
[] sh:path [ sh:zeroOrMorePath rdf:rest ] ;
   sh:minCount 2 .
```

A Dict item follows the same rules as a Dict in an ordinary slot. If it has
a stored `spec`, it must fit the item type, including structural matching for a
sugar type. Its fields must satisfy the requested item type in either case.
For example, given `items: List<of:Addr>` and `Addr: Dict { city: Str }`,
this item fails because it lacks `city`:

```trio
items: [{id:@home}]
```

The field shape checks each item along the list. The two alternatives allow an
inferred type or a declared type that fits Addr. The final constraint checks
Addr's fields in both cases.

```turtle
[] sh:path ( [ sh:zeroOrMorePath rdf:rest ] rdf:first ) ;
   sh:nodeKind sh:BlankNode ;
   sh:or (
     [ sh:property [ sh:path sys:spec ; sh:maxCount 0 ] ]
     [ sh:class ex:Addr ]
   ) ;
   sh:node ex:Addr-fields .
```

For example, `[{id:@home city:"Here"}]` supplies an inline Addr. Supplying
`[@home]` instead gives the list a Ref item and fails even when `@home`
resolves to a valid Addr.

For a sugar item type, the second alternative uses the same fitting predicate
as a [nested sugar slot](ObjectSlots.md#nested-sugar-types). A scalar item
uses `sh:datatype`.

Lists of enum values, Choice selections, units, references, or other lists are
not supported in this version. Exporting one of these list forms is an error;
see [Deferred Mappings](Deferred.md).

# Instance Data

A Xeto list becomes an RDF collection, preserving item order.
For `List<of:Addr>`, each dictionary item becomes an inline blank node at
`rdf:first`. The item uses the same field and ID mapping as a dictionary
stored in an ordinary slot. See [Object Slots](ObjectSlots.md#instance-data).

```xeto
@batch1: Batch {
  names: { "alpha", "beta" }
}
```

```turtle
ex:batch1 a xetoRdf:Record, ex:Batch ;
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

When an override exports one stored list through two slot properties, both
properties point to the same list head. For example, if Register narrows
`Directory.items` from `List<of:Addr>` to `List<of:Special>`, the stored field

```trio
items: [{spec:@example::Special city:"Here" note:"ok"}]
```

produces these links:

```turtle
ex:register ex:Directory.items _:items ;
            ex:Register.items  _:items .
_:items rdf:first [ a ex:Special ; sys:spec ex:Special ;
                    ex:Addr.city "Here"^^xsd:string ; ex:Special.note "ok"^^xsd:string ] ;
        rdf:rest rdf:nil .
```

An empty list uses `rdf:nil` for both properties.
