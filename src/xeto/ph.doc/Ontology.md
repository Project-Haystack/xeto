<!--
title:      Ontology
author:     Brian Frank
created:    17 Jul 2020
copyright:  Copyright (c) 2020, Project-Haystack
-->

# Overview

Project Haystack's semantic model is structured into three layers:

  - **Vocabulary**: gives every tag a formal definition via globals
  - **Taxonomy**: organizing our terms into a subtype tree
  - **Ontology**: modeling more complex relationships between definitions

Each standardized tag name used by Haystack data has a [global](doc.xeto::Globals)
definition in the [ph::PhEntity] spec.  This ensures that all Haystacks use a
consistent [controlled vocublary](https://en.wikipedia.org/wiki/Controlled_vocabulary).

These tags are organized into a [taxonomy](https://en.wikipedia.org/wiki/Taxonomy)
tree of type specs that organizes concepts from most general to most specific via
the mechanism of spec subtyping.  For example, the [ph::ElecMeter] spec is a
subtype of the more general [ph::Meter] spec.

The suite of all `"ph.*"` libraries, specs, and inter-relationships
define the [ontology](https://en.wikipedia.org/wiki/Ontology_(information_science)).
Adhering to the ontology enables interoperable data models to be shared
across applications.

# Instance Models

The ontology defines the *meta model* which is used to model *concepts*.  We
use the term *instance model* when we build a Haystack data model for
specific buildings and systems.  One good way to think of instances is as
proper nouns with a unique names and identities.  For example, the Empire
State Building is an instance of a site, as opposed to the [ph::Site] type
which models the concept of all buildings.

# Entities

We use the term *entity* to describe a unique instance in a Haystack model.
Entities model things from the real world like buildings, rooms, equipment,
and sensors.  An entity in Haystack is always modeled as a [dict](Kinds#dict)
(collection of tags).

The [sys::Entity] taxonomy tree defines the fundamental types used to build Haystack
data models. We define how to model the following fundamental entities of the
built environment:

- [Site](Sites.md): single building with its own street address
- [System](Systems.md): logical group of equipment
- [Space](Spaces.md): location or zone within a site
- [Equip](Equips.md): physical or logical piece of equipment within a site
- [Point](Points.md): sensor, actuator or setpoint for an equip
- [WeatherStation](Weather.md): weather station observations
- [Device](Devices.md): computers, controllers, networking gear

Each of these entity types is discussed in detail in the following chapters.

All entities are uniquely identified via the [sys::Entity.id] tag.  The `id` tag
serves as the primary key and must be unique within the scope of the entity's dataset.
We use the `id` tag to cross-reference our relationships between entities.
For example, spaces and equipment contained within a given site will model
their containment relationship via the [ph::PhEntity.siteRef] tag.

Entities should always be given a [ph::PhEntity.dis] tag that provides a human friendly
name of the entity.  A general rule is that display names should be relatively
short (under 40 characters), but also fully descriptive of the entity.

