# Overview

Sugar specs are convenience names for a nominal type plus a set of structural
typing constraints.  The term borrows from *syntax sugar* in programming
languages: syntax that makes code sweeter to write but adds nothing that could
not be expressed without it.

They are declared with the `sugar` marker meta:

```xeto
DischargeFanRunCmd: DuctFanRunCmd <sugar> { discharge }
```

This defines `DischargeFanRunCmd` as a name for "a `DuctFanRunCmd` with
the `discharge` marker".  Any instance that matches `DuctFanRunCmd` and
has the `discharge` tag matches `DischargeFanRunCmd` - whether or not
the instance references `DischargeFanRunCmd` by name.

Sugar specs are eliminable vocabulary: they add names, not semantics.
Every sugar spec can be mechanically replaced by its nominal anchor plus
its constraints with no change in meaning.  Sugar specs are never
required for semantic interoperability; the nominal specs and tags alone
carry the semantics.

# Motivation

Ontology concepts frequently vary along orthogonal dimensions.  A fan
run command is one concept along the *function* axis, but the fan may
sit in the discharge, return, or exhaust duct section; it may serve the
cold or hot deck; it may be stage one or stage two of a fan array.
These dimensions multiply.  Modeling every combination as its own
nominal spec explodes the type system: 100 fan and damper points across
nine duct sections and three decks is thousands of specs, hand-authored
and always incomplete.

Sugar specs factor the model instead:

- The nominal hierarchy carries one axis: function.  `DuctFanRunCmd` is
  a nominal spec, and its `ductSection` choice slot declares the
  dimension.
- The dimension values are ordinary tags on the instance: `discharge`,
  `coldDeck`, `stage`.  [Choices](Choices.md) ensure coherent
  selection.
- Combinations that need names get them as sugar one-liners.
  Combinations that don't remain fully queryable as filters.

No combination is ever enumerated up front, yet every combination can be
queried and validated on demand as anonymous sugar - named sugar specs are
minted only where a handle is useful, never to make a combination usable.

# Syntax

A sugar spec is any spec with the [sys::Spec.sugar] marker in its meta.  The
marker is inherited, so every subtype of a sugar spec is itself sugar:

```xeto
DischargeFanRunCmd: DuctFanRunCmd <sugar> { discharge }

// sugar is inherited from DischargeFanRunCmd
ColdDeckDischargeFanRunCmd: DischargeFanRunCmd { coldDeck }

// invariant scalar constraint
Stage2DischargeFanRunCmd: DischargeFanRunCmd { stage: Int <invariant> 2 }
```

Because the marker is inherited, the nominal/sugar boundary is crossed
at most once down any inheritance chain and only in one direction: a
nominal spec may not extend a sugar spec.

# Body Rules

Every slot in a sugar spec's body must resolve to a
[global slot](Globals.md) thru the inheritance chain of the spec's
type.  Each slot is exactly one of:

- **Marker constraints**: `discharge` - a required marker; the instance
  must have the tag
- **Invariant constraints**: `stage: Int <invariant> 2` - a scalar the
  instance must equal; scalars are matched by equality only
- **Defaults**: `unit: "cfm"`, `bacnetCurAddr: { addr: "AI3" }` -
  scalar and nested values used by instantiation
- **Queries**: query slots that constrain the graph, such
  as required points on an equip; see [Queries](#queries) below

Only markers and invariants are constraints.  The distinction is
syntactic: a bare marker or a slot with the `invariant` meta constrains
membership; any other value is a default and never participates in
matching or filters:

```xeto
NaturalGasFlowSp: FluidVolumetricFlowSp <sugar> {
  naturalGas          // constraint
  unit: "cfm"         // default
}
```

Any fluid volumetric flow setpoint with the `naturalGas` tag matches
`NaturalGasFlowSp` whatever its unit; instantiating it defaults unit to `cfm`.

A sugar spec may *not* declare new structural slots.  Declaring
structure is nominal-spec work; a sugar spec only names, constrains,
and defaults.  The following are compile errors:

```xeto
Bad1: DuctFanRunCmd <sugar> { surgeMargin: Number }  // not a global tag
Bad2: DuctFanRunCmd <sugar> { dischrage }            // not a global tag
Bad3: DuctFanRunCmd <sugar> { stage: Int }           // neither constraint nor default
Bad4: DuctFanRunCmd <sugar> { discharge: Marker? }   // optional marker constrains nothing
Bad5: DuctFanRunCmd <sugar> { *newTag: Marker }      // declares a global
```

Because every name must resolve, misspelled tags are compile errors
rather than silently new slots.  Invariants follow the normal rule
that a subtype cannot redeclare an invariant value, so conflicting
scalar constraints down a chain are compile errors.  Two markers which
select different options of one exclusive choice, such as `{discharge,
return}`, are reported by [choice](Choices.md) validation.

# Nominal Anchor

Every sugar spec must bottom out at exactly one nominal spec, called
its *nominal anchor*.  The spec's type may be a nominal spec, another
sugar spec, or an `And` of specs - but after flattening, the nominal
ancestors must reduce to a single most specific anchor:

```xeto
// legal: both operands anchor on DuctFanRunCmd
X: DischargeFanRunCmd & Stage2DischargeFanRunCmd <sugar>

// compile error: two unrelated nominal anchors
Y: Foo & Bar <sugar>
```

If two concepts genuinely cross-cut, the cross-cutting dimension
belongs in a tag, not a second anchor.  `Or` and maybe types are not
permitted in sugar specs; a sugar spec is always a pure conjunction of
one anchor plus constraints.

At compile time every sugar spec is flattened to its *nominal anchor*
and its *effective constraints* (the union of constraints down the
chain, canonically sorted).  For example `ColdDeckDischargeFanRunCmd`
above flattens to anchor `DuctFanRunCmd` with constraints `{discharge,
coldDeck}`.  Note the flattening stops at the nominal anchor:
`DuctFanRunCmd`'s own body tags such as `fan` and `air` do *not* become
matching constraints.  For a nominal spec, body tags are necessary
conditions checked by validation, never sufficient conditions for
membership.

# Matching

An instance matches a sugar spec if either:

1. its `spec` tag nominally asserts the sugar spec (or a subtype), or
2. it matches the nominal anchor and satisfies every effective
   constraint

Membership is the union of what the instance asserts and what its tags
prove.  This is the same rule as nominal specs with the second clause
added; for a nominal spec the second clause contributes nothing.

```xeto
// all of these match DischargeFanRunCmd
{spec: @ph.points::DuctFanRunCmd, discharge}   // by anchor + tags
{spec: @acme::DischargeFanRunCmd, discharge}   // by assertion (and tags)
{spec: @acme::DischargeFanRunCmd}              // by assertion (invalid - see below)
```

Matching tests only the anchor and constraints.  It never checks the
anchor's slot conformance and never evaluates queries - those are
validation concerns.  An instance missing a required slot or a required
point is an *invalid* member, not a non-member; it still matches
queries for its type, which is essential for finding and fixing
malformed data.  See [Constraints](Constraints.md) for validation.

Subtyping between sugar specs is computed: `A` is a subtype of `B` iff
A's anchor is a subtype of B's anchor and A's constraints are a
superset of B's.

# Filters

A sugar spec name in a filter denotes its computed extension.  The
name lowers at compile time into primitives the filter engine already
executes:

```
DischargeFanRunCmd
```

is equivalent to:

```
DuctFanRunCmd and discharge
```

(plus the nominal assertion clause described above).  There is no
structural evaluator at runtime; after lowering, the filter consists
only of is-a tests and tag tests, so existing indexes and query
planning apply unchanged.

Filters also accept anonymous sugar syntax - a spec name followed by a
constraint body:

```
DuctFanRunCmd { discharge, coldDeck }
```

This is an anonymous sugar spec: same body rules, same lowering, no
name.  It lets any tag combination be queried in type form whether or
not a named sugar spec was ever minted for it.  The braces admit
only constraints (markers and scalar equality); richer
predicates such as ranges compose outside the braces with `and`.

# Instances

The canonical instance form asserts the most specific nominal spec and
carries the dimensions as tags:

```xeto
{spec: @ph.points::DuctFanRunCmd, discharge, stage: 2}
```

Instances may also assert a sugar spec directly in their `spec` tag.
Instantiation stamps the spec's effective constraints and defaults
onto the instance, and validation requires the constraints to be present even
when the name is asserted - the name never substitutes for the tags.
An instance that asserts a sugar spec but is missing or contradicting
its constraint tags is invalid.

When data is exported for interchange, tools should normalize asserted
sugar specs to the canonical form so that sugar names never travel on
the wire.  This preserves the rule that sugar libs are never required
for interoperability.

# Queries

Sugar spec bodies may declare query slots.  Queries constrain the
graph neighborhood of an instance - for example the points required on
an equip:

```xeto
StandardVav: Vav <sugar> {
  singleDuct
  points: {
    DischargeAirTempSensor
    DischargeFanRunCmd
  }
}
```

Queries participate only in validation, never in matching or filters.
An instance matches `StandardVav` based on anchor and tags alone;
whether it actually has the required points is checked by validation.
This keeps filter evaluation free of graph traversal and preserves the
rule that broken instances remain findable as what they are.

Queries make sugar specs suitable as named validation profiles: a
shareable, versioned spec describing the required shape of an
installation that any party can publish over the core ontology.

Each item in a query body is itself an anonymous sugar spec: its
declared type is the anchor and its body follows the same
[body rules](#body-rules).  Named and anonymous items are equivalent,
and an item may carry defaults for instantiation and tooling:

```xeto
points: {
  DischargeFanRunCmd                // named sugar
  DuctFanRunCmd { discharge }       // same constraints, anonymous
  zoneTemp: ZoneAirTempSensor {     // no constraints beyond the type
    dis: "Zone Temp"                // default
    bacnetCurAddr: { addr: "AI1" }  // default
  }
}
```

Validation matches each item against the query's extent by anchor and
constraints only; defaults such as `dis` and protocol addresses are
never matched.

# Mixins

[Mixins](Mixins.md) may target sugar specs, subject to the same body
rules: a mixin on a sugar spec may add constraints, defaults, and
queries, but not structural slots, so that the extended spec remains a
valid sugar spec:

```xeto
// legal: adds a constraint and a query requirement
+StandardVav {
  hotWaterHeating
  points: {
    HotWaterValveModulatingCmd
  }
}

// compile error: structural slot on a sugar spec
+StandardVav {
  installNotes: Str
}
```

Note that mixin constraints narrow the sugar spec's extension within
the namespaces that include the mixin's lib.  As with all mixins, the
extended spec is namespace dependent.

# Sugar Libs

By convention, sugar specs for a core ontology library are distributed
in a companion library with the `sugar` suffix.  For example the sugar
vocabulary over `ph.points` is published as `ph.points.sugar`.

The convention carries a contract:

- A sugar lib contains only sugar specs; the core lib contains only
  nominal specs
- A sugar lib is a conservative extension: eliminating it changes no
  semantics, only names
- A sugar lib is optional; conforming implementations must not require
  it for interoperability
- Core libs never depend on sugar libs; shapes in core libs use
  anonymous sugar syntax instead of sugar names

Sugar names are minted on demand - when documentation, common queries,
or validation profiles need a handle - never speculatively to
enumerate a product space.  Unnamed combinations cost nothing: they
remain fully expressible as anonymous sugar in filters and slot types.

# Design Rationale

The split between `ph.points` and `ph.points.sugar` illustrates these
rules.  The principles used to draw that line generalize to any core/sugar
lib pair.

**Structure lives on nominal specs.**  A dimension enters the model as
a choice slot, and sugar bodies cannot declare new slots - so every
constrained dimension needs a nominal home.  `DuctAirTempSensor`
declares `ductSection: DuctSection` and `ductDeck: DuctDeck?`;
`PipeWaterTempSensor` declares `pipeSection: PipeSection`; the fluid
point bases narrow `pointSubject` to a required `Fluid`.  Sugar then
names points along the dimension - `DischargeAirTempSensor`,
`HotWaterValveModulatingCmd` - and its constraint markers
simultaneously satisfy matching and the choice validation.

**A required choice replaces abstract.**  The choice-gated bases are
deliberately concrete.  A `DuctAirTempSensor` cannot validly exist
without selecting a duct section, which makes it effectively abstract -
but it must remain assertable, because the canonical instance form is
the base plus tags: `{spec: @ph.points::DuctAirTempSensor, discharge}`.
Marking it abstract would leave sugar names as the only route into the
extension, breaking the rule that sugar is eliminable.

**Which concepts get nominal names.**  A spec must be nominal when
other nominal specs extend it, since a nominal spec may not extend
sugar: `AirTempSensor` and `WaterFlowSensor` are nominal because the
`Duct*` and `Pipe*` layers anchor on them.  A subject marker with no
choice machinery and no combinatorics is a first-class concept rather
than sugar: `ZoneAirTempSensor`, `ParkingBayOccupiedSensor`,
`WindowOpenSensor` are nominal.  Sugar earns its keep only where a
choice-validated dimension or a combinatorial space needs names -
`naturalGas` over the fluid choice, `discharge` over the duct section
choice.  Beyond these tests, a core ontology may deliberately grant
first-class nominal treatment to important vocabulary, as `ph.points`
does for the air, water, steam, and refrigerant point families.

# Versioning

A sugar spec's extension is determined by its definition at a pinned
lib version.  Adding a constraint to a published sugar spec narrows
its extension and is a breaking change, exactly as adding a required
slot to a nominal spec is.  The non-breaking way to tighten is to
publish a new, more specific sugar spec alongside the old one.
Adding or changing a default leaves the extension unchanged and is not
a breaking change.

