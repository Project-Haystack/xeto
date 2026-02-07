<!--
title:      Spaces
author:     Brian Frank
created:    13 Apr 2021
copyright:  Copyright (c) 2021, Project-Haystack
-->

# Overview
Spaces are modeled with the [ph::PhEntity.space] marker tag.  Spaces include the physical
structure of buildings including [floors](ph::PhEntity.floor) and [rooms](ph::PhEntity.room).  Spaces
also encompass logical system-oriented [zones](ph::ZoneSpace) for HVAC, lighting,
security, smoke/fire, etc.

# Tags
All spaces must define a [ph::PhEntity.siteRef] tag for their parent site.  In
addition, when spaces are wholly contained by another space, they should
model that relationship via the [ph::PhEntity.spaceRef] tag.

# Floors
If a building has multiple floors, then each floor should be modeled using
the [ph::PhEntity.floor] and [ph::PhEntity.space] marker tags.  Floors are numbered using the [ph::PhEntity.floorNum]
tag using the European convention where the ground floor is always floor zero.
Subterranean floors are numbered using negative numbers.  Here are examples
for the proper modeling of floors:

    // ground floor; typically called first floor in the US
    id: @site.floor1
    dis: "Ground Floor"
    ground
    floor
    space
    floorNum: 0
    siteRef: @site

    // basement floor
    id: @site.basement
    dis: "Basement Level 1"
    subterranean
    floor
    space
    floorNum: -1
    siteRef: @site

    // roofs are modeled as a special floor
    id: @site.roof
    dis: "Roof level of two story building"
    roof
    floor
    space
    floorNum: 2
    siteRef: @site

# Zones
Zones are system oriented spaces.  Currently we define the following zone
types:
  - [ph::HvacZoneSpace]: heating/cooling/ventilation zones
  - [ph::LightingZoneSpace]: lighting zones

Here is an example of an HVAC zone:

    id: @site.zone3B
    dis: "VAV Zone 3-B"
    hvac
    zone
    space
    siteRef: @site
    spaceRef: @site.floor3

Zones, and their associated points, are examined in detail in the [Zones] chapter.

# Data Centers
Data centers are also modeled as a subtype of space.
See the [DataCenters] chapter for details.
