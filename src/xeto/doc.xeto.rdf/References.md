# Ref and MultiRef Slots

## SHACL Validation

### Ordinary Reference Targets

A `Ref` maps to an IRI identifying its target. With `of`, the target must fit
the specified type. Ordinary types use RDF class membership. A sugar target
accepts either a matching type or a compatible base type with fields satisfying
its sugar requirements. Without `of`, the target must be a `sys:Entity`, so an
IRI with no corresponding record does not satisfy the constraint.

```xeto
Site : Dict

Equip : Dict {
  siteRef: Ref<of:Site>
  related: MultiRef?<of:Equip>
  auxRef: Ref?
}
```

```turtle
ex:Equip a sh:NodeShape ;
  sh:targetClass ex:Equip ;
  sh:property [
    sh:path ex:Equip.siteRef ;
    sh:nodeKind sh:IRI ;
    sh:class ex:Site ;
    sh:minCount 1 ;
    sh:maxCount 1 ;
  ] ;
  sh:property [
    sh:path ex:Equip.related ;
    sh:nodeKind sh:IRI ;
    sh:class ex:Equip ;
  ] ;
  sh:property [
    sh:path ex:Equip.auxRef ;
    sh:nodeKind sh:IRI ;
    sh:class sys:Entity ;
    sh:maxCount 1 ;
  ] .
```

Every value of a `MultiRef` must pass the same reference checks as a single `Ref`.

### Query Body Member Types

A record declared as a [Query body member](Queries.md#query-body-validation) can satisfy
a typed reference through inheritance. For example:

```xeto
Target: Dict {}
Twins: Dict {
  next: MultiRef?
  results: Query<of:Target, via:"next"> {
    first: Target
  }
}
Holder: Dict { targetRef: Ref<of:Target> }
```

The `first` declaration creates the type `Twins.results.first`, which derives
from Target. A record declared as that type can therefore be the value of
Holder's `targetRef`. This is an ordinary reference check; it does not require
running Query-body validation.

The exporter declares the member's class and nominal inheritance in the
validation vocabulary:

```turtle
ex:Twins.results.first a rdfs:Class ;
  rdfs:subClassOf ex:Target .
```

The reference therefore uses the ordinary class constraint:

```turtle
sh:class ex:Target
```

An unchanged inherited member retains the declaring member's identity. An
override has its own member identity and exports its effective nominal base.
These declarations describe types; they do not run Query-body validation.

### Sugar Reference Targets

To validate `Ref<of:Hot>`, the SHACL constraint checks whether the referenced
record fits Hot. [Type matching](Mappings.md#type-matching-and-field-validation)
can accept either a matching declared type or a structural sugar match; it does
not validate all of Hot's fields. For example:

```xeto
Target: Dict { *hot: Marker, *label: Str }
Hot: Target <sugar> { hot, label: Str "default" }
Owner: Dict {
  results: Query<of:Target, via:"next"> {
    wanted: Target { hot }
  }
}
Holder: Dict { targetRef: Ref<of:Hot> }
```

`Owner.results.wanted` fits `Hot` because both require `hot`. It does not
inherit from `Hot`, so it does not inherit the `label` field. A `Ref<of:Hot>`
must therefore accept records of this type without requiring `label`:

```turtle
sh:or (
  [ sh:class ex:Hot ]
  [ sh:class ex:Owner.results.wanted ]
)
```

Keep this alternative in the reference's validation shape. Under the current
validation mapping, do not export this computed relationship as
`rdfs:subClassOf ex:Hot`: that statement would also cause Hot's validation shape
to check the member records and require `label`.

The target record can also fit Hot through its fields. For example, this runtime
record declares Target, rather than Hot, but contains the required `hot` tag:

```trio
id: @sensor
spec: @example::Target
hot
```

A `Ref<of:Hot>` accepts this record even though `label` is absent. Reference
fitting checks Hot's sugar requirements; it does not run Hot's full validation
shape. The reference constraint therefore combines two alternatives:

```text
fits Hot = membership in a type that fits Hot
           OR (membership in Hot's anchor, Target
               AND presence of hot)
```

The first alternative includes computed sugar matches such as
`Owner.results.wanted`. The second checks the referenced record's anchor type
and the presence of `hot`. Both alternatives belong to the reference's SHACL
constraint; neither requires executing a Query. The following excerpt shows
that constraint inside Holder's shape. Unrelated shape declarations are omitted:

```turtle
ex:Holder a sh:NodeShape ;
  sh:targetClass ex:Holder ; # Check records declared as Holder or its subtypes.
  # Other shape declarations omitted.
  sh:property [
    sh:path ex:Holder.targetRef ; # The targetRef slot declared above.
    sh:nodeKind sh:IRI ;         # A Ref is represented by an IRI.
    sh:minCount 1 ;              # targetRef is required.
    sh:maxCount 1 ;              # Ref holds one value, unlike MultiRef.
    sh:or (
      [ sh:class ex:Hot ] # A declared Hot type establishes fitting.
      [ sh:class ex:Owner.results.wanted ] # This member type also fits Hot.
      [ sh:class ex:Target ; # Structural matching first requires Hot's anchor.
        sh:or ( # Then check whether hot is present, in any stored value form.
          [ sh:property [
              # An ordinary value stored under a property for the hot tag.
              sh:path [ sh:alternativePath ( ex:Target.hot ex:Hot.hot ) ] ;
              sh:minCount 1
          ] ]
          [ sh:property [
              sh:path sys:hasMarker ; # Evidence that hot stores a Marker.
              # Count only evidence naming a property for hot.
              sh:qualifiedValueShape [ sh:in ( ex:Target.hot ex:Hot.hot ) ] ;
              sh:qualifiedMinCount 1
          ] ]
          [ sh:property [
              sh:path sys:hasRefField ; # Ref evidence also preserves empty MultiRefs.
              # Count only evidence naming a property for hot.
              sh:qualifiedValueShape [ sh:in ( ex:Target.hot ex:Hot.hot ) ] ;
              sh:qualifiedMinCount 1
          ] ]
        )
      ]
    )
  ] .
```

The property lists are abbreviated. A translation resolves every property for
`hot` in the selected namespace, including Query-member properties. The three
presence checks cover ordinary values, Markers, and explicitly stored empty
MultiRefs. Presence does not require the stored value to be a Marker.
Invariant requirements instead compare the stored value with the required
value. The [invariant comparison rules](Queries.md#sugar-matching) also apply
to reference fitting; the output here is a SHACL constraint, not a Query.

Apply these fitting rules to every `MultiRef` target and to reusable result
constraints. Keep fitting predicates local to the reference shape; do not add
computed subclass relationships or apply the target's full validation shape.

## Instance Data

A `Ref` becomes one object IRI. A `MultiRef` becomes one triple per referenced
IRI using the same property.

```xeto
Site : Dict

Equip : Dict {
  siteRef: Ref<of:Site>
  related: MultiRef?<of:Equip>
}

@site1: Site {}
@equip2: Equip { siteRef: @site1 }
@equip3: Equip { siteRef: @site1 }

@equip1: Equip {
  siteRef: @site1
  related: MultiRef {
    @equip2
    @equip3
  }
}
```

```turtle
ex:site1 a sys:Entity, ex:Site .

ex:equip2 a sys:Entity, ex:Equip ;
  ex:Equip.siteRef ex:site1 ;
  sys:hasRefField ex:Equip.siteRef .

ex:equip3 a sys:Entity, ex:Equip ;
  ex:Equip.siteRef ex:site1 ;
  sys:hasRefField ex:Equip.siteRef .

ex:equip1 a sys:Entity, ex:Equip ;
  ex:Equip.siteRef ex:site1 ;
  ex:Equip.related ex:equip2, ex:equip3 ;
  sys:hasRefField ex:Equip.siteRef, ex:Equip.related .
```

The single `siteRef` value produces one statement. The two `related` values
produce two statements with the same subject and property. Referenced Xeto
entities are exported with their own `sys:Entity` and concrete spec types so
typed and untyped reference shapes can validate them.

### Reference-field metadata

RDF uses IRIs for Refs, Units, UnitQuantity values, and named nested Dicts.
A link alone does not show which Xeto value produced it. `sys:hasRefField`
identifies a Ref or MultiRef field on the record that stores it; the value of
`sys:hasRefField` is the field's property IRI.

| Stored Xeto value | Xeto Query behavior |
| --- | --- |
| Ref to an existing record | Follow it and consider that record for results. |
| Ref to a missing record | Return no result from that branch. |
| Unit, UnitQuantity, or named nested Dict | Fail if the Query tries to follow it as a Ref. |

Without this metadata, SPARQL could return an empty result for the last case
where Xeto reports an error. Query input validation uses `sys:hasRefField` to
preserve that distinction.

A Ref contains the ID of another record. That referenced record need not be
present in the dataset. In library `example.people`, the metadata belongs to
the record holding the Ref (`ex:` denotes that library's RDF namespace):

The following RDF excerpts show instance data; schema triples are omitted.

```xeto
Person: Dict { address: Ref }
@person1: Person { address: Ref "example.people::missingAddress" }
```

```turtle
ex:person1 a sys:Entity, ex:Person ;
  ex:Person.address ex:missingAddress ;
  sys:hasRefField ex:Person.address .
# No record for ex:missingAddress is present in this graph.
```

The graph contains the IRI `ex:missingAddress`, but no record describing that
address, so this branch returns no result. `sys:hasRefField` describes the Ref
stored on person1, not whether the address record exists. Separate record
validation may reject the dangling Ref.

Before following a field with values, Query input validation requires its
`sys:hasRefField` statement. For example, following `address` on person1 above
passes this check; removing the statement makes it fail. This prevents a nested
Dict or a Unit IRI such as `unit:PA` from being mistaken for a Ref. An absent
field or an empty MultiRef has nothing to follow and needs no statement.
Consumers must preserve these statements when copying or transforming RDF.

Exporters use the stored value to decide whether to add `sys:hasRefField`,
including for `Obj` slots and undeclared fields. If a field is exported under
several property IRIs, such as inherited aliases, the exporter lists each
property IRI as a value of `sys:hasRefField` on the owning record.
For example, an override narrows an inherited Ref to an Address Ref. The
export retains both property IRIs and identifies both as reference fields:

```xeto
Address: Dict {}
Person: Dict { address: Ref }
Employee: Person { address: Ref<of:Address> }
@address1: Address {}
@employee1: Employee { address: @address1 }
```

```turtle
ex:address1 a sys:Entity, ex:Address .
ex:employee1 a sys:Entity, ex:Employee ;
  ex:Person.address ex:address1 ;
  ex:Employee.address ex:address1 ;
  sys:hasRefField ex:Person.address, ex:Employee.address .
```

For an explicitly stored empty MultiRef, the exporter lists the field's
property IRI under `sys:hasRefField`, but there are no referenced IDs to export.
An absent field produces neither reference values nor `sys:hasRefField` metadata:

```xeto
Group: Dict { members: MultiRef? }
@group1: Group { members: MultiRef {} }
```

```turtle
ex:group1 a sys:Entity, ex:Group ;
  sys:hasRefField ex:Group.members .
# No ex:Group.members values: the MultiRef is empty.
```

