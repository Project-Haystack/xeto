<!--
title:      Equips
author:     Brian Frank
created:    13 Apr 2021
copyright:  Copyright (c) 2021, Project-Haystack
-->

# Equips
Equipment assets are modeled with the [ph::Equip] spec.  Equipment is often
a physical asset such as an air handler, boiler, chiller, or meter. However,
equip can also be used to model a logical grouping such as an entire chilled
water plant.

# Tags
Equip entities must define a [ph::PhEntity.siteRef] tag for their parent site.
Additionally, use the [ph::PhEntity.spaceRef] tag if the equipment is contained by a defined
space.  It is also typical to nest equipment using containment via the [ph::PhEntity.equipRef]
tag.  For example, if a fan is contained by an AHU, then the fan entity should
define an `equipRef` that references the parent AHU entity.

# Examples
Here are examples for the proper tagging of equipment:

    // electric meter for a building
    id: @site.mainMeter
    dis: "Main Elec Meter"
    ac
    elec
    meter
    equip
    siteMeter
    siteRef: @site
    spaceRef: @site.utility-closest

    // chilled water plant
    id: @site.chillerPlant
    dis: "Chiller Plant"
    chilled
    water
    plant
    equip
    siteRef: @site

    // chiller contained by plant from previous example
    id: @site.chiller3
    dis: "Chiller-3"
    chiller
    equip
    siteRef: @site
    equipRef: @site.chillerPlant

# Points
The standardized points for equip are:

 - [ph.points::EnableCmd]: command that permits or prohibits equip to operate
 - [ph.points::RunCmd]: commands an equip to run or stop running
 - [ph.points::RunSensor]: senses the on/off state of an equip
 - [ph.points::RuntimeSensor]: senses equipment runtime

For heating equip the standardized points are:

 - [ph.points::HeatEnableCmd]: command that permits/prohibits heating equip to run
 - [ph.points::HeatRunCmd]: commands heating equip on/off
 - [ph.points::HeatRunSensor]: senses on/off state of heating equip
 - [ph.points::HeatModulatingCmd]: commands modulating heating capacity
 - [ph.points::HeatModulatingSensor]: senses modulating heating capacity

Both [ph.points::HeatRunCmd] and [ph.points::HeatRunSensor] have an optional [ph::PhEntity.stage] tag that should be applied to points for heating equipment that operate at discrete stages.

Modulating heating capacity is expressed as a percentage, ranging from 0% (no capacity) to 100% (full capacity).

Similarly for cooling equip the standardized points are:

 - [ph.points::CoolEnableCmd]: command that permits/prohibits cooling equip to run
 - [ph.points::CoolRunCmd]: commands cooling equip on/off
 - [ph.points::CoolRunSensor]: senses on/off state of cooling equip
 - [ph.points::CoolModulatingCmd]: commands modulating cooling capacity
 - [ph.points::CoolModulatingSensor]: senses modulating cooling capacity

Both [ph.points::CoolRunCmd] and [ph.points::CoolRunSensor] have an optional [ph::PhEntity.stage] tag that should be applied to points for cooling equipment that operate at discrete stages.

Modulating cooling capacity is expressed as a percentage, ranging from 0% (no capacity) to 100% (full capacity).
