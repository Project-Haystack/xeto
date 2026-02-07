<!--
title:      Devices
author:     Brian Frank
created:    13 Apr 2021
copyright:  Copyright (c) 2021, Project-Haystack
-->

# Overview
We use the term *device* to describe any asset that is microprocessor based.
Typically, this includes most IT and networking gear, but also includes
microprocessor based controllers commonly used to perform control and monitoring
of equipment.

There can often be a fuzzy distinction between equipment versus devices,
especially since most point data for an equipment is manifested through a
controller.  However when building a detailed model of a control system,
use an [ph::PhEntity.equip] entity to represent the physical asset and use a separate [ph::PhEntity.device]
entity to represent the controller.  In cases where the distinction is
not clear, then combine the [ph::PhEntity.equip] and [ph::PhEntity.device] tags into one entity.

# Networks
Networks are modeled with the [ph::PhEntity.network] tag.  Devices on a network should
model their network relationship with the [ph::PhEntity.networkRef] tag.

Devices where the main function is networking should subtype from
the [ph::NetworkingDevice] conjuct.  Haystack defines the following networking
device types:

  - [ph::NetworkingRouter]: device used to route data between different networks
  - [ph::NetworkingSwitch]: device used to connect devices on the same network

Here is a simple example of a network model that includes a switch
with two controllers:

    id: @my-network
    dis: "Simple Local Network"
    network

    id: @my-switch
    dis: "Network Switch"
    networking
    switch
    device
    networkRef: @my-network

    id: @my-controller-1
    dis: "Controller 1"
    controller
    device
    networkRef: @my-network

    id: @my-controller-2
    dis: "Controller 2"
    controller
    device
    networkRef: @my-network
