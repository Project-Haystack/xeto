<!--
title:      Zones
author:     Brian Frank
created:    20 Apr 2021
copyright:  Copyright (c) 2021, Project-Haystack
-->

# Overview
This chapter provides the details for modeling zones and their associated
sensors and setpoints.  See the [Spaces#zones] chapter for an introduction
to how zones are modeled as a subtype of space.

# Occupancy
Determining and controlling occupancy is one of the most critical aspects
of building automation.  Haystack defines several occupancy related tags
and points:

  - [ph::PhEntity.occupied] [ph::PhEntity.sp]: setpoint true when occupied, false when unoccupied
  - [ph::PhEntity.occupied] [ph::PhEntity.sensor]: boolean sensor true when occupied, false when unoccupied
  - [ph::PhEntity.occupancy] [ph::PhEntity.sensor]: people counter that measures number of occupants
  - [ph::PhEntity.occ]: marker tag is used on points for occupancy modes
  - [ph::PhEntity.unocc]: marker tag is used on points for occupancy modes
  - [ph::PhEntity.occupants]: the people who occupy a space

The primary tag used for the occupied/unoccupied state is the [ph::PhEntity.occupied] tag.
In most cases, the occupied setpoint is a schedule that determines occupancy
based on the time of day and day of the week.  We might also use a sensor
like a motion sensor to determine occupancy. More sophisticated sensors can
actually count the number of people in a space; in which case, we use the [ph::PhEntity.occupancy]
tag.  As a general rule: points with the [ph::PhEntity.occupied] tag should have
a `Bool` kind; points with the [ph::PhEntity.occupancy] tag should have a `Number` kind.

The [ph::PhEntity.occ] and [ph::PhEntity.unocc] tags are used in cases when we need to distinguish
modes.  For example, if we have two different temperature setpoints based
on the occupancy mode, then we distinguish them as `occ temp sp`
and `unocc temp sp`.

# HVAC
The [ph::HvacZoneSpace] conjunct models an HVAC zone for the conditioning
of space comfort and air quality.

The sensor and setpoints associated with temperature control include:

  - [ph::PhEntity.hvacMode] [ph::PhEntity.sp]: current mode such as "cooling" or "heating"
  - [ph::PhEntity.temp] [ph::PhEntity.sensor]: actual sensed temperature of the space
  - [ph::PhEntity.temp] [ph::PhEntity.occ] [ph::PhEntity.cooling] [ph::PhEntity.sp]: cooling setpoint when occupied
  - [ph::PhEntity.temp] [ph::PhEntity.occ] [ph::PhEntity.heating] [ph::PhEntity.sp]: heating setpoint when occupied
  - [ph::PhEntity.temp] [ph::PhEntity.unocc] [ph::PhEntity.cooling] [ph::PhEntity.sp]: cooling setpoint when unoccupied
  - [ph::PhEntity.temp] [ph::PhEntity.unocc] [ph::PhEntity.heating] [ph::PhEntity.sp]: heating setpoint when unoccupied
  - [ph::PhEntity.temp] [ph::PhEntity.standby] [ph::PhEntity.cooling] [ph::PhEntity.sp]: cooling setpoint when in standby mode
  - [ph::PhEntity.temp] [ph::PhEntity.standby] [ph::PhEntity.heating] [ph::PhEntity.sp]: heating setpoint when in standby mode
  - [ph::PhEntity.temp] [ph::PhEntity.effective] [ph::PhEntity.sp]: current setpoint we are controlling to taking
    into account cooling/heating mode and occ/unocc/standby mode

Whenever possible, there should be one effective temperature setpoint
that takes all the various modes into account.  This provides the simplest
model to perform analysis of HVAC operations.  However in some cases
a thermostat will provide two effective setpoints - one for cooling
and one for heating.  In this case, there must also be a `hvacMode` point
to determine which one is the true effective setpoint.  That setup should
look like this:

  - [ph::PhEntity.hvacMode] [ph::PhEntity.sp]
  - [ph::PhEntity.temp] [ph::PhEntity.effective] [ph::PhEntity.cooling] [ph::PhEntity.sp]
  - [ph::PhEntity.temp] [ph::PhEntity.effective] [ph::PhEntity.heating] [ph::PhEntity.sp]

In addition, we might also find the following points:

  - [ph::PhEntity.pressure] [ph::PhEntity.sensor]: measured static pressure of the space
  - [ph::PhEntity.pressure] [ph::PhEntity.sp]: static pressure setpoint (commonly used for lab situations)
  - [ph::PhEntity.humidity] [ph::PhEntity.sensor]: measured relative humidity of the space
  - [ph::PhEntity.humidity] [ph::PhEntity.sp]: setpoint for relative humidity
  - [ph::PhEntity.dewPoint] [ph::PhEntity.sensor]: measured dew point temperature of the space
  - [ph::PhEntity.dewPoint] [ph::PhEntity.sp]: setpoint for dew point temperature
  - [ph::PhEntity.enthalpy] [ph::PhEntity.sensor]: measured heat content of the space

All the points above must also be tagged with [ph::PhEntity.zone], [ph::PhEntity.air], and [ph::PhEntity.point].

# Air Quality
It is also common in an HVAC zone to also measure and control air quality.
Typical air quality points include:

  - [ph::Ch2oConcentration] [ph::PhEntity.sensor]: measured formaldehyde (CH₂O)
  - [ph::CoConcentration] [ph::PhEntity.sensor]: measured carbon monoxide (CO)
  - [ph::Co2Concentration] [ph::PhEntity.sensor]: measured carbon dioxide (CO₂)
  - [ph::Co2Concentration] [ph::PhEntity.sp]: configured max carbon dioxide (CO₂)
  - [ph::Nh3Concentration] [ph::PhEntity.sensor]: measured ammonia (NH₃)
  - [ph::No2Concentration] [ph::PhEntity.sensor]: measured nitrogen dioxide (NO₂)
  - [ph::O3Concentration] [ph::PhEntity.sensor]: measured ozone (O₃)
  - [ph::Pm01Concentration] [ph::PhEntity.sensor]: measured particulate matter 0.1
  - [ph::Pm25Concentration] [ph::PhEntity.sensor]: measured particulate matter 2.5
  - [ph::Pm10Concentration] [ph::PhEntity.sensor]: measured particulate matter 10
  - [ph::TvocConcentration] [ph::PhEntity.sensor]: measured total volatile organic compounds

All the points above must also be tagged with [ph::PhEntity.zone], [ph::PhEntity.air], and [ph::PhEntity.point].

# Lighting
The [ph::LightingZoneSpace] conjuct models lighting zones.  Typical points used
for lighting measurement and control include:

  - [ph::LightLevel] [ph::PhEntity.sensor]: brightness level status as percentage
  - [ph::LightLevel] [ph::PhEntity.sp]: brightness level setpoint as percentage
  - [ph::PhEntity.light] [ph::PhEntity.illuminance] [ph::PhEntity.sensor]: lux, footcandle, or phot
  - [ph::PhEntity.light] [ph::LuminousFlux] [ph::PhEntity.sensor]: luminous flux in lumens

