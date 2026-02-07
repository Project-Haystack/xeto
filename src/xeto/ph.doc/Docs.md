<!--
title:      Docs
author:     Brian Frank
copyright:  Copyright (c) 2019, Project-Haystack
license:    Licensed under the Academic Free License version 3.0
-->

# Overview
This chapter provides guidance on how to use the reference documentation.
Each section used in the documentation is described below.

# Index
The main index page is composed of the following sections:

## Taxonomies
This section lists key terms with a link to an index of the term's
subtype tree.  You can add new terms into this section with the [ph::PhEntity.docTaxonomy]
tag.

## Listings
This section contains links to miscellaneous listing pages including:
  - `tags`: all terms composed of an atomic name
  - `conjuncts`: all terms composed of multiple tags
  - `libs`: listing of library modules
  - `filetypes`: listing of Haystack filetype encodings

## Tags
Section within a lib index that lists the terms defined by the lib
module (both atomic tags and conjuncts).

# Manuals
Manuals are hand written documents that provide design details
to augment the auto-generated reference documentation.  Manuals
are composed of chapters that correspond to individual HTML pages.

A manual index is composed of two sections:
  - manual: summary information including name and brief overview
  - chapters: index of the manual's chapter pages

# Defs
Each def has its own reference page in the documentation composed
of the following sections:

## Def
The `def` section contains summary information for the definition
including its identifier formatted [doc] description.

## Conjunct
This section is used only for [conjuncts](Defs#conjuncts).  It
provides a link to the definition of each of the conjunct's parts.

## Meta
The `meta` section documents the [normalized](Normalization)
tags for the definition's representation as a dict.  Each tag
present in the definition is listed here, although some values such as
the `doc` or `enum` may reference another section.

## Usage
This section is used only on entity terms.  It lists the required
marker tags that must be applied to a dict to correctly
[implement](Reflection#implementation) the term.  It takes into
account conjuncts and supertypes with the [mandatory] marker.

## Enum
If the definition has the [ph::PhEntity.enum] tag, then this section provides
a listing of the documented enumerations and their summary.

## Supertypes
This section lists the term's [supertypes](Subtyping).  The supertypes
are organized as an indented tree down from most general to most specific.
The first item is the root type, which is typically [sys::Marker] or [ph::PhEntity.val].  The
last item will be the definition's declared supertype matching the [is] tag.
If the term has multiple supertypes, then this tree will have multiple leaves.

## Subtypes
This section lists the term's [subtypes](Subtyping).  The subtypes
are organized in an indented tree structure.

## Tags
This section lists the value tags associated with the entity via
the [tagOn] association.  This section also includes tags that are
registered on the entity's supertypes.

## Children
This section lists children [prototypes](Protos) that are "contained" by
the given entity type.

# Protos
Prototypes are dict instances that model a predefined collection of tags.

## Proto
The `proto` section defines the prototypes tags.

## Implements
The definitions [implemented](Reflection#implementation) by the prototype.
