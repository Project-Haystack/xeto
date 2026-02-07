<!--
title:      VAVs
author:     Brian Frank
created:    26 Feb 2011
copyright:  Copyright (c) 2011, Project-Haystack
license:    Licensed under the Academic Free License version 3.0
-->

# Overview
The [ph::PhEntity.vav] tag is used to model variable air volume terminal units.  VAVs
are supplied conditioned air from an [AHU](AHUs) and use a damper to
modulate the air flow to the zone.  Typically VAVs are supplied cool
air at a temperature of around 55°F (13°C).  Some VAVs support *reheat*
that allow them to heat this cool air to a warmer temperature.

A VAV without reheat must be defined with the [ph::PhEntity.coolingOnly] tag. Otherwise, it
must define one of the [heating process](#heating-process) tags.  Some reheat
VAVs contain a fan, in which case they should also define the [ph::PhEntity.fanPowered] tag.

VAVs should define the [ph::PhEntity.airRef] tag to reference their supply AHU.  When
there are multiple supply AHUs, then the `airRef` tag should be a list
of all the upstream AHUs.

# Choices
AHUs define a suite of [choices](Choices) that should be made on
a per instance basis.

## Heating Process
If the VAV provides reheat, then it should add one of the following
[ph::HeatingProcess] choices:
  - [ph::PhEntity.elecHeating]
  - [ph::PhEntity.hotWaterHeating]
  - [ph::PhEntity.steamHeating]

Or if the VAV does not support reheat, then add the [ph::PhEntity.coolingOnly] tag.

When applicable the heating choice marker should also be paired with
the appropriate flow ref tag.  For example, if a VAV uses hot water
for reheat, then it should also define the [ph::PhEntity.hotWaterRef] tag to refer
to its hot water plant.

## Modulation
One of the following [ph::VavModulation]
markers should be defined:
  - [ph::PhEntity.pressureDependent]
  - [ph::PhEntity.pressureIndependent]

## vavAirCircuitType
If the VAV is fan powered, then one of the following [ph::VavAirCircuit]
markers should be defined:
  - [parallel](vav-parallel)
  - [series](vav-series)

## Ductwork
One of the following [ph::DuctConfig] markers should be
defined:
  - [ph::PhEntity.singleDuct]
  - [ph::PhEntity.dualDuct]

# Sections
We associate points with sections of a VAV using these tags:
  - [ph::PhEntity.inlet]: air entering the unit from the AHU
  - [ph::PhEntity.discharge]: air exiting the unit to be supplied to the zones
  - [ph::PhEntity.zone]: conditioned space associated with the unit

Since most points are not clearly associated with the inlet or discharge
ducts, we typically omit a section tag for most points. However, any
points that would conflict with the zone points must be
qualified with either `inlet` or `discharge`.

# Examples
The following are examples of fully tagged VAVs:

    // Cooling only VAV
    id: @vav
    dis: "VAV-Example"
    vav
    equip
    coolingOnly
    pressureDependent
    singleDuct
    airRef: @ahu
    siteRef: @site

    // Fan powered VAV with hot water reheat
    id: @vav
    vav
    equip
    fanPowered
    hotWaterHeating
    pressureDependent
    singleDuct
    hotWaterRef: @hot-water-plant
    airRef: @ahu
    siteRef: @site
