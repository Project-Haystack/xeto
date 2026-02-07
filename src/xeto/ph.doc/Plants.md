<!--
title:      Plants
author:     Brian Frank
created:    22 Apr 2021
copyright:  Copyright (c) 2021, Project-Haystack
license:    Licensed under the Academic Free License version 3.0
-->

# Overview
This chapter covers plants used to produce thermal energy for cooling or
heating.  Haystack standardizes the following types of plants:

  - [ph::ChilledWaterPlant]: produces chilled water for cooling
  - [ph::HotWaterPlant]: produces hot water for heating
  - [ph::SteamPlant]: produces steam water

Typically, these plants supply [AHUs] and [VAVs] for HVAC
applications.  But they can also be used for industrial processes.

We typically use the term *central plant* to describe a plant that
supplies a building or group of buildings.  When the plant supplies
a municipality, we use the term *district plant*.  However, the Haystack
modeling techniques described here are identical.

Haystack models an entire plant as a single blackbox [ph::PhEntity.equip] entity.
Equipment within the plant, including heat exchangers, pumps, boilers,
and chillers, are modeled as children of the plant via the [ph::PhEntity.equipRef] tag.

# Chilled Water Plants
Chilled water plants are modeled as a single black box using the [ph::ChilledWaterPlant]
conjunct along with the [ph::PhEntity.equip] tag.  These plants are composed of
the following types of sub-equipment:
  - [ph::PhEntity.chiller]
  - [ph::PhEntity.coolingTower]
  - [ph::PhEntity.heatExchanger]
  - [ph::PumpMotor]
  - [ph::ValveActuator]

This diagram shows the salient equipment and fluid flows:

![](chilled-water-plant.svg)

# Hot Water Plants
Hot water plants are modeled using the [ph::HotWaterPlant] conjunct along
with the [ph::PhEntity.equip] tag.  These plants are composed of the following types
of sub-equipment:
  - [ph::HotWaterBoiler]
  - [ph::PhEntity.heatExchanger]
  - [ph::PumpMotor]
  - [ph::ValveActuator]

# Steam Plants
Steam plants are modeled using the [ph::SteamPlant] conjunct along
with the [ph::PhEntity.equip] tag.  These plants are composed of the following types
of sub-equipment:
  - [ph::SteamBoiler]
  - [ph::PhEntity.heatExchanger]
  - [ph::PumpMotor]
  - [ph::ValveActuator]

# Loops
Plants typically maintain a separation between the working fluid within
the plant versus the fluid that supplies thermal energy to the building.
We term the pipework within the central plant the [ph::PhEntity.primaryLoop] and
the pipework from the plant to the building the [ph::PhEntity.secondaryLoop].  These
two loops are connected through [heat exchangers](ph::PhEntity.heatExchanger).  In
the case of a third loop connected to the second loop through a second
set of heat exchangers we use the term [ph::PhEntity.tertiaryLoop].

For example, here is how we would model a heat exchanger with a temperature
sensor on the primary entering pipe and on the secondary leaving pipe:

    id: @hx
    dis: "Heat Exchanger"
    chilled
    water
    heatExchanger
    equip
    equipRef: @plant
    siteRef: @site

    id: @primary-temp
    dis: "Primary Entering Temp Sensor"
    chilled
    water
    primaryLoop
    entering
    temp
    sensor
    point
    unit: "°C"
    kind: "Number"
    equipRef: @hx
    siteRef: @site

    id: @secondary-temp
    dis: "Secondary Leaving Temp Sensor"
    chilled
    water
    secondaryLoop
    leaving
    temp
    sensor
    point
    unit: "°C"
    kind: "Number"
    equipRef: @hx
    siteRef: @site
