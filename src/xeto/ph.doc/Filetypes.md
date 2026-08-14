<!--
title:      Filetypes
author:     Brian Frank
created:    17 Jul 2020
copyright:  Copyright (c) 2020, Project-Haystack
-->

# Overview
Haystack defines several text formats for encoding the fixed set
of standard [data types](Kinds).  Each format is modeled by a file spec
such as [sys.files::ZincFile], which carries its mime type and file
extensions.

# Zinc
Zinc is a recursive acronym for "Zinc Is Not CSV".  It is the original
Haystack format designed to encode CSV with strong typing.  Zinc provides
full fidelity to encode all Haystack kinds without loss of typing.  It
provides a compact and readable syntax at the expense of requiring a non-trivial
custom parser.  Zinc's scalar encodings are also the basis for Trio and JSON.
Zinc is the default format used for the [HttpApi] (although today JSON is
also equally supported and utilized).

See [Zinc] chapter for further discussion and grammar.

# Hayson
Hayson is the JSON encoding of Haystack data.  It maps the Haystack data
types to JSON without loss of information by tagging each typed scalar with
a `_kind` discriminator.  There are two versions:
 - [Version 4](Hayson#json-version-4) - The default JSON encoding for Haystack
 - [Version 3](Hayson#json-version-3) - The Haystack 3 encoding for JSON. This encoding
 is supplanted by the version 4 encoding.  It is still supported for backwards
 compatibility, but is deprecated and will be removed in a future version.

See [Hayson] chapter for further details.

Hayson is not the only JSON encoding in use: [Jeto](doc.xeto::Jeto) is
xeto-typed JSON, where the spec supplies the types instead of each value
declaring its own kind.  Both are JSON, so a file extension of "json" does not
say which one a file holds.

# Trio
Trio is an acronym for Tag Record Input/Output.  Trio is derived from
[YAML](https://en.wikipedia.org/wiki/YAML).  It uses the Zinc encodings
for scalar types to provide full type fidelity.  Trio is targeted for use
cases when humans need to hand code Haystack data.  Most examples in the
documentation are formatted in Trio.

Trio is a line oriented format.  Dicts are encoded with each tag on its own
line.  Dicts are separated by a line of "---".  There is also some syntax sugar
for multi-line strings and nested collection data values.

Trio is not ideal for representing grids because it does not support grid
meta nor column meta.  As such, Trio should not typically be used to encode
requests/responses in the [HttpApi].

See [Trio] chapter for further details.

# CSV
CSV stands for Comma Separated Values.  CSV files are easily imported and
exported from spreadsheets and relational databases.  CSV is the inspiration
for Zinc.  The main drawback with CSV is that there is essentially only one
collection type (table/grid) and one scalar type (strings).  As such CSV
does not provide full fidelity with Haystack kinds.  However as a widely
supported open format, we specify a standard mechanism to export Haystack
data to CSV.

See [Csv] chapter for further details.

# RDF
Project Haystack specifies a standard export to RDF triples via two different
formats: Turtle and JSON-LD.  Both defs and instance data have a standard export
mapping.

See [Rdf](doc.xeto::Rdf) chapter for details.

