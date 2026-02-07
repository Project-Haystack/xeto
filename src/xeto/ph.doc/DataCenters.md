<!--
title:      Data Centers
author:     Brian Frank
created:    14 Apr 2021
copyright:  Copyright (c) 2021, Project-Haystack
-->

# Overview
Haystack defines the fundamentals for modeling data centers via the [ph::DataCenter]
and [ph::Rack] specs.  Data centers are considered a specialized type of [ph::Space].
Equipment and devices contained within the data center define their relationship
to the data center via the [ph::PhEntity.spaceRef] tag.

# Racks
Racks are modeled as specialized [ph::Equip] with the [ph::Rack] marker tag.  The
servers and networking gear contained by the rack declare their containment
relationship via the [ph::PhEntity.equipRef] tag.

# Examples
Here is a simple example for a data center containing one rack that contains
one server:

    id: @dc
    dis: "Simple Data Center"
    dataCenter
    space
    siteRef: @site

    id: @rack1
    dis: "Rack-1"
    rack
    equip
    spaceRef: @dc
    siteRef: @site

    id: @server-1A
    dis: "Rack-1 Server-A"
    server
    computer
    device
    rackRef: @rack1
    spaceRef: @dc
    siteRef: @site

