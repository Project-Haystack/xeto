<!--
title:      Equips
author:     Brian Frank
created:    13 Apr 2021
copyright:  Copyright (c) 2021, Project-Haystack
-->

# Equips
Equipment assets are modeled with the [ph::PhEntity.equip] marker tag.  Equipment is often
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
