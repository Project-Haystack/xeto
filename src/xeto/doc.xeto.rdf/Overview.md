# Overview

Xeto defines a standard mapping of specs and instance data to RDF 1.1. The
mapping uses RDFS for vocabulary relationships, SHACL Core for validation, and
QUDT identifiers for units and quantity kinds.

This chapter defines three profiles. A profile is a group of related mapping
rules that an exporter can support:

- the **vocabulary profile** maps Xeto specs and metadata to RDF resources;
- the **SHACL profile** maps Xeto validation semantics to SHACL shapes; and
- the **instance profile** maps Xeto instances and values to RDF data.

An exporter can support one or more profiles. A single Turtle file may contain
statements from all three profiles, or an exporter may write each profile to a
separate file. The profiles do not need to be separated into named graphs
inside an RDF dataset.

**Profile applicability.** Each construct later in this chapter includes only
the profile subsections that apply to it. If a construct has no `Vocabulary`,
`SHACL Validation`, or `Instance Data` subsection, that profile adds no separate
RDF statements for the construct. The omission is intentional; it does not mean
that the mapping is unfinished. Mappings that are not yet supported are listed
explicitly under [Deferred Mappings](Deferred.md#deferred-mappings).

# Conformance

The Xeto project maintains an implementation of this mapping for normal RDF
export. This chapter calls it the **reference exporter**. It supports all three
profiles, but this specification defines conformance; the reference exporter's
exact Turtle output does not.

An RDF export conforms to:

- the **vocabulary profile** when it contains every required vocabulary
  statement for each spec included in the export;
- the **SHACL profile** when it contains every required shape and constraint
  for each effective spec included in the export; and
- the **instance profile** when it contains every required data statement for
  each instance included in the export.

Although this specification aims to cover all Xeto constructs, some do not yet
have a mapping; see [Deferred Mappings](Deferred.md#deferred-mappings). Conformance is
fail-closed, an exporter must reject an unsupported construct rather than
silently omit it or emit weaker RDF or SHACL.

Another exporter may support any combination of these profiles and may include
additional descriptive RDF statements. Those additions must not conflict with
this mapping or change which instance data passes validation. For behavior
covered by this specification, output of additional SHACL constraints that reject
data accepted by Xeto, or omission of constraints that accept data rejected
by Xeto, do not conform to this specification.

The same RDF graph can be written as Turtle in several different ways. For
example, `ph:Equip` and `haystack:Equip` identify the same resource when both
prefixes expand to the same full IRI. Triple order, whitespace, and the local
labels assigned to blank nodes also do not change the graph's meaning. For this
reason, exporters are compared using the parsed RDF graph and SHACL validation
results rather than their Turtle text. Likewise, RDF-consuming applications
should therefore parse the RDF graph instead of comparing Turtle files line
by line.

This chapter defines what RDF an exporter produces, but it does not prescribe
how a user asks for that output. One tool might write vocabulary, SHACL shapes,
and instance data into one Turtle file. Another might provide separate commands
or options for producing three files. Both approaches conform when each
requested profile contains the statements defined here. No particular command
name, option name, file layout, or named-graph layout is required.

# Non-Goals

This mapping does not require:

- OWL inference;
- stored inverse relationship triples;
- byte-for-byte Turtle compatibility between exporters;
- copying complete external ontologies into every export; or
- a production dependency between Haxall and any independent conformance tool.

SPARQL query generation and optional relationship materialization are separate
concerns from RDF vocabulary, SHACL validation, and instance projection.

