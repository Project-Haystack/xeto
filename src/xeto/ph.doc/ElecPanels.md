<!--
title:      Elec Panels
author:     Brian Frank
created:    12 Aug 2014
copyright:  Copyright (c) 2014, Project-Haystack
-->

# Overview
This chapter covers the modeling of electrical panels, circuits, and
electricity flows.

# Elec Panel
Electrical panels are modeled by the [ph::ElecPanel] conjunct.

A panel should define the [ph::PhEntity.elecRef] to reference its upstream source.  A
site level panel will probably reference the main site meter.  Sub-panels
typically reference their parent panel via `elecRef`.  In any case,
the `elecRef` tag should always reference the most direct upstream source
of electricity.

Here is an example of a electrical panel directly downstream of the
main site meter:

    id: @main-elec-panel
    dis: "Main Elec Panel"
    elec
    panel
    equip
    elecRef: @main-meter
    siteRef: @site

# Circuit
The [ph::PhEntity.circuit] tag models an individual electrical circuit.  Circuits are used
to model meta-data about the electrical circuit such as max load, phase, etc.
Circuits also contain the points associated with sensors and breakers.  Note
these tags have not yet been formalized by Project Haystack.

Circuits should reference their source panel via the [ph::PhEntity.elecRef] tag.  The panel
is where the circuit's fuse or breaker is located.

Here is a simple example for an electrical circuit in the panel from above:

    id: @circuit-hallway
    dis: "Hallway Electrical Circuit"
    circuit
    equip
    elecRef: @main-elec-panel
    siteRef: @site
    spaceRef: @hallway


