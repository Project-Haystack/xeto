<!--
title:      Motors
author:     Brian Frank
created:    12 Aug 2014
copyright:  Copyright (c) 2014, Project-Haystack
-->

# Overview
The [ph::PhEntity.motor] tag is paired with the [ph::PhEntity.equip] tag to model equipment that
operates with an electric motor.  Haystack defines the following
standardized motor subtypes:

  - [ph::FanMotor]: fans used to move air
  - [ph::PumpMotor]: pumps used to move fluid

Often these motors are driven by a variable frequency drive, or VFD, that
exposes many points.  If a motor uses a VFD, then it should also be tagged
with the [ph::PhEntity.vfd] marker.  Typically, motors such as fans and pumps are
sub-components of a larger piece of equip that we nest via
the [ph::PhEntity.equipRef] tag.

# Points
The standardized points for motor control are:

  - [ph::PhEntity.run] [ph::PhEntity.cmd]: primary on/off command
  - [ph::PhEntity.enable] [ph::PhEntity.cmd]: secondary on/off command
  - [ph::PhEntity.run] [ph::PhEntity.sensor]: run status sensor
  - [ph::PhEntity.enable] [ph::PhEntity.sensor]: enable status sensor
  - [ph::PhEntity.alarm] [ph::PhEntity.sensor]: bolean alarm condition

The primary on/off point of equipment is always modeled with the [ph::PhEntity.run] tag.
Paired with [ph::PhEntity.cmd] it models the on/off command point; paired with [ph::PhEntity.sensor] it
models the run status point.  Many VFDs also include a secondary [ph::PhEntity.enable] point
that requires both `run` and `enable` to be commanded to true in order for the
equipment to be on.  In all cases, `true` models the *on* state, and `false` models
the *off* state.

If the motor is driven by a variable frequency drive, then it should be tagged
with the [ph::PhEntity.vfd] marker.  Points related to the drive speed control:

  - [vfd speed](ph::VfdSpeed) [ph::PhEntity.cmd]: speed command as percentage where 0% is off, 100% if full speed
  - [vfd freq](ph::VfdFreq) [ph::PhEntity.cmd]: speed command as a frequency in Hz
  - [vfd speed](ph::VfdSpeed) [ph::PhEntity.sensor]: speed status as percentage where 0% is off, 100% if full speed
  - [vfd freq](ph::VfdFreq) [ph::PhEntity.sensor]: speed status as frequency in Hz

Many VFDs will also provide many of the same points as an electric meter.
Measurements such as electric demand, consumption, voltage, and current
should follow the same conventions as [elec meters](Meters#electric-meters).

# Fans
Fans may optionally be defined as either an [ph::PhEntity.equip] or a [ph::PhEntity.point].  If the
fan motor is driven by a VFD, then it is recommended to make the fan a sub-equip.
However in many cases a simple fan in a terminal unit such as a [ph::PhEntity.vav]
is more easily modeled as just a [ph::PhEntity.point].

## Fan Points
In simple cases where the fan is just a command and/or feedback sensor,
then it is best to model it as a [ph::PhEntity.point].

If annotated as an output with the [ph::PhEntity.cmd] tag, then the point
models the command status of the fan:
  - false (off) or true (on)
  - variable speed is 0% (off) to 100% (full speed)

If annotated as an input with the [ph::PhEntity.sensor] tag, then the
point models a sensor used to verify the fan status:
  - false indicates no air flow (off) or true indicates successful
    airflow (fan is on)
  - if numeric, the point is differential pressure across the fan
    measured in "inH₂O" or "kPa"

## Fan Equips
When the fan motor is a VFD, it should be modeled as an [ph::PhEntity.equip] entity
using the standard VFD points described above. If you wish to standardize
modeling all fans as equip, then simple single speed fans should define
their state via a [ph::PhEntity.run] point.

Example of a VFD fan on an AHU:

    id:@ahu ahu equip
    id:@ahu-fan equipRef:@ahu discharge fan vfd motor equip
    id:@ahu-fan-run    equipRef:@ahu-fan discharge fan run cmd point
    id:@ahu-fan-status equipRef:@ahu-fan discharge fan run status point
    id:@ahu-fan-speed  equipRef:@ahu-fan discharge fan drive speed cmd unit:"%" point

Note that the fan is modeled as an sub-equip of the AHU via the [ph::PhEntity.equipRef]
tag.  The VFD points are defined under the fan itself, however we must
*flatten* the `discharge` and  `fan` tags into the points.

# Pumps
Pumps may optionally be defined as either an [ph::PhEntity.equip] or a [ph::PhEntity.point].  If
the pump is a VFD, then it is recommended to make it an equip level entity.
However, if the pump is modeled as a simple on/off point as a component within
a large piece of equipment such as a [ph::PhEntity.boiler], then it is modeled as just a
[ph::PhEntity.point].  Pumps should follow the same point and equip level modeling
conventions as [fans](#fans).

