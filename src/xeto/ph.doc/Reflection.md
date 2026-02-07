<!--
title:      Reflection
author:     Brian Frank
created:    25 Mar 2019
copyright:  Copyright (c) 2019, Project-Haystack
-->

# Overview
Defs define our meta-model and dicts define our data instances.
Haystack provides a standardized processes to map between
the two:
   - *implementation*: mapping defs to a dict using the appropriate tags
   - *reflection*: mapping a dict to its effective defs

We say a dict *implements* a def when the it contains the appropriate
tags.  For example if a dict defines the [ph::PhEntity.equip] tag, then the dict
implements the [ph::PhEntity.equip] def.  If a dict contains both the [ph::PhEntity.hot] and [ph::PhEntity.water]
tags then the dict implements [ph::HotWater].

Reflection is the inverse process.  We are given a dict and we compute
the list of effective defs that the it implements.

Implementation and reflection in Haystack are a flavor of
[structural typing](https://en.wikipedia.org/wiki/Structural_type_system).
Dict data never explicitly declares itself of a single class or type.
Instead we use a combination of marker tags to indicate typing information.
We use value tags to indicate an attribute or fact about the data.

# Implementation
The rules for a dict to implement a def are as follows:

  1. Based on the def type:
      a. Tag def: the single tag name is applied
      b. Conjunct: each tag within the conjunct is applied
      c. Feature keys: are never implemented

  2. We walk the supertype tree of the def and apply any
     tag that is marked as `mandatory`

Lets look at some examples.

If we want to apply the [ph::PhEntity.point] def to a dict, then we apply rule 1a.  There
are no supertypes marked as `mandatory,` so we are done.  Thus, we apply
only `{point}` to fully implement the [ph::PhEntity.point] def.

If we wish to apply the [ph::PhEntity.rtu] def, then we apply rule 1a.  However, when
we walk the supertype tree, we see that it subtypes from two `mandatory`
defs: [ph::PhEntity.ahu] and [ph::PhEntity.equip].  Therefore, to implement [ph::PhEntity.rtu] requires adding
all three tags: `{rtu, ahu, equip}`.  As a general rule, top-level entity
markers such as [ph::PhEntity.site], [ph::PhEntity.space], [ph::PhEntity.equip], [ph::PhEntity.point] are always required.
This makes it easy to query your tag based data.

If we wish to apply the [ph::HotWaterPlant] def, then we will apply rule 1b.
Additionally, this def subtypes from the mandatory [ph::PhEntity.equip] tag.  So we need to
add four tags `{hot, water, plant, equip}`.

# Inheritance
When a dict implements a def, it implicitly implements all of that def's
supertypes.  For example if we add the [ph::PhEntity.water] tag to a dict, then we
implicitly implement all of the supertypes of [ph::PhEntity.water], which include
[ph::Liquid], [ph::Fluid], [ph::Substance], [ph::Phenomenon], and [sys::Marker].  We call
this flattened list of supertypes the def's *inheritance*.

# Reflection
Reflection is the inverse of implementation.  Given a dict, we compute
the defs it implements.  Reflection is always done within the context
of a [namespace](Namespaces) to determine which defs are in scope.

To reflect a dict to its def, we walk each tag in the dict and
apply the following rules:
  1. Check if the tag name maps to a tag def
  2. If the tag maps to a possible conjunct, then check if
     the dict has all the conjunct's tags
  3. Infer the [inheritance](#inheritance) from all defs reflected
     from the previous steps

Lets follow this process to reflect the following dict:

    id: @hwp, dis: "Hot Water Plant", hot, water, plant, equip

Step 1 yield: [sys::Entity.id], [ph::PhEntity.dis], [ph::PhEntity.hot], [ph::PhEntity.water], [ph::PhEntity.plant], [ph::PhEntity.equip]

Step 2 yields: [ph::HotWater], [ph::HotWaterPlant]

Step 3 yields:
  - from hot-water-plant/equip: [sys::Entity], [sys::Marker]
  - from id/dis: [sys::Ref], [sys::Str], [ph::PhEntity.scalar], [ph::PhEntity.val]

# Entity Types
We use markers extensively in Haystack to indicate typing information.  However,
a marker tag is not a *type* per se.  In a tagging system, we use tags to
quickly and easily associate data to related terms.  Let's examine our
example from above:

    id: @hwp, dis: "Hot Water Plant", hot, water, plant, equip

This dict implements the term [ph::HotWater], but we would not say that
the plant "is-a" type of hot-water.  Those tags only mean that this data
is related to hot-water.  This makes it convenient to query our data for
things related to hot-water, water, or liquid.

However, we would say that the dict above "is-a" type of [ph::HotWaterPlant].  We can
reflect this information because [ph::HotWaterPlant] is a subtype of [sys::Entity].
The `entity` subtype tree models primary typing information.  Terms that
subtype from entity must be designed to stand on their own - they cannot be used
in conjuncts.
