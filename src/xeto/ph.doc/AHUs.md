<!--
title:      AHUs
author:     Brian Frank
created:    26 Feb 2011
copyright:  Copyright (c) 2015, Project-Haystack
license:    Licensed under the Academic Free License version 3.0
-->

# Overview
We define a wide range of equipment for air handling including:
  - [ph::PhEntity.ahu]: air handling unit
  - [ph::PhEntity.doas]: dedicated outside air system
  - [ph::PhEntity.mau]: makeup air unit
  - [ph::PhEntity.rtu]: roof top unit
  - [ph::PhEntity.fcu]: fan coil unit
  - [ph::PhEntity.crac]: computer room air conditioner
  - [ph::PhEntity.unitVent]: unit ventilator
  - [ph::PhEntity.heatPump]: heat pump

All of these equipment types follow the same fundamental modeling
techniques and subtype from [ph::AirHandlingEquip].

# Choices
AHUs define a suite of [choices](Choices) that should be made on
a per instance basis.

## Heating Process
If the AHU provides heating, then it should add one of the following
[ph::HeatingProcess] choices:
  - [ph::PhEntity.dxHeating]
  - [ph::PhEntity.elecHeating]
  - [ph::PhEntity.hotWaterHeating]
  - [ph::PhEntity.naturalGasHeating]
  - [ph::PhEntity.steamHeating]

Or if the unit does not provide heating, then add the [ph::PhEntity.coolingOnly] tag.

When applicable the heating choice marker should also be paired with
the appropriate flow ref tag.  For example if an AHU uses hot water
heating, then the AHU should also define the [ph::PhEntity.hotWaterRef] tag to refer
to its hot water plant.

## Cooling Process
If the AHU provides cooling, then it should add one of the following
[ph::CoolingProcess] choices:
  - [ph::PhEntity.chilledWaterCooling]
  - [ph::PhEntity.dxCooling]

Or if the unit does not provide cooling, then add the [ph::PhEntity.heatingOnly] tag.

If using chilled water, then the [ph::PhEntity.chilledWaterRef] should be defined
to reference the chilled water plant.

## Air Volume Adjustability
One of the following [ph::AirVolumeAdjustability]
markers should be defined:
  - [ph::PhEntity.constantAirVolume]
  - [ph::PhEntity.variableAirVolume]

This choice models the AHU's ability to adjust the volume of air flow.
Typically this distinction is based on whether the AHU's fan is single
speed or a VFD.

## Zone Delivery
One of the following [ph::AhuZoneDelivery] choices
should be defined to model how the AHU delivers conditioned air to
the zone:

  - [ph::PhEntity.vavZone]: AHU supplies air to VAV terminal units

  - [ph::PhEntity.directZone]: AHU supplies air directly to the zone

  - [ph::PhEntity.chilledBeamZone]: AHU supplies air to chilled beam terminal units

  - [ph::PhEntity.multiZone]: air is split into a duct per zone

A Variable Volume Temperature or VVT system is defined as a constant volume
AHU with VAV terminal units.  This is indicated by the presence of both
the [ph::PhEntity.constantAirVolume] and [ph::PhEntity.vavZone] tags.

## Ductwork
One of the following [ph::DuctConfig] markers should be
defined:

  - [ph::PhEntity.singleDuct]: AHU uses a single duct

  - [ph::PhEntity.dualDuct]: the AHU discharges to two ducts, which is some combination
    of `hotDeck`, `coldDeck`, or `neutralDeck`

  - [ph::PhEntity.tripleDuct]: the AHU discharges into three ducts, which are the [ph::PhEntity.hotDeck],
    `coldDeck`, and `neutralDeck`

# Sections
Most points in an AHU are associated with one of the following
[ph::DuctSection] of the unit:
  - [ph::PhEntity.discharge]: air exiting the unit to be supplied to the zones/terminal units
  - [ph::PhEntity.return]: air returning from the zone back into the unit
  - [ph::PhEntity.outside]: fresh, outside air entering the unit for air quality and economizing
  - [ph::PhEntity.exhaust]: air exiting the unit back outside
  - [ph::PhEntity.mixed]: return and outside air mixed together before passing
    through the heating/cooling elements
  - [ph::PhEntity.cool]: cooling elements/coils
  - [ph::PhEntity.heat]: heating elements/coils
  - [ph::PhEntity.zone]: conditioned space associated with the unit

If the AHU has one duct used for both fresh air and economizing then use the
[ph::PhEntity.outside] tag.  If there are two dedicated ducts, then use the [ph::PhEntity.ventilation]
and [ph::PhEntity.economizing] tags for the respective ducts.

The follow diagram shows the AHU sections and logical flow of air:

![](ahu-sections.svg)


# Examples
The following are examples of fully tagged AHUs:

    // Built up AHU supplied by central plants with VAV zones
    id: @abc
    dis: "AHU"
    ahu
    equip
    hotWaterHeating
    chilledWaterCooling
    vavZone
    singleDuct
    siteRef: @xyz

    // Rooftop unit direct to zones
    id: @abc
    dis: "RTU-2"
    rtu
    ahu
    equip
    elecHeating
    dxCooling
    constantAirVolume
    directZone
    singleDuct
    siteRef: @xyz

