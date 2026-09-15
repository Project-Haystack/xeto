<!--
title:      Motors
author:     Brian Frank
created:    12 Aug 2014
copyright:  Copyright (c) 2014, Project-Haystack
-->

# Overview
The [ph::PhEntity.motor] tag is paired with the [ph::PhEntity.equip] tag to model equipment that
operates with an electric motor.  Haystack defines the following entities
that use motors:

  - [ph::Fan]: fans used to move air
  - [ph::Pump]: pumps used to move fluid

Often these motors are driven by a variable frequency drive, or VFD, that
exposes many points.  If a motor uses a VFD, then it should also be tagged
with the [ph::PhEntity.vfd] marker.  Typically, motors such as fans and pumps are
sub-components of a larger piece of equip that we nest via
the [ph::PhEntity.equipRef] tag.

# Points
Standardized points for equip are extended to motors as follows:

  - [ph.points::MotorEnableCmd]: command that permits or prohibits a motor to operate
  - [ph.points::MotorRunCmd]: commands a motor to run or stop running
  - [ph.points::MotorRunSensor]: senses the on/off state of a motor
  - [ph::PhEntity.alarm] [ph::PhEntity.sensor]: boolean alarm condition

Permission from [ph.points::MotorEnableCmd] may be required for [ph.points::MotorRunCmd] to
take effect.

If the motor is driven by a Variable Frequency Drive (VFD), then it should be tagged
with the [ph::PhEntity.vfd] marker.

Points related to motor speed control:

  - [ph.points::MotorSpeedModulatingCmd]: command for modulating motor speed
  - [ph.elec::VfdOutputElecAcFreqCmd]: command for a VFD's output AC electric frequency
  - [ph.points::MotorSpeedModulatingSensor]: sensor for modulating motor speed
  - [ph.elec::VfdOutputElecAcFreqSensor]: sensor for a VFD's output AC electric frequency

Modulating speed is defined as a percentage where 0% is off and 100% is full speed.

Many VFDs will also provide many of the same points as an electric meter.
Measurements such as electric demand, consumption, voltage, and current
should follow the same conventions as [elec meters](Meters#electric-meters).

# Fans
Fans may optionally be defined as either an [ph::PhEntity.equip] or a [ph::PhEntity.point].  If the
fan motor is driven by a VFD, then it is recommended to make the fan a sub-equip.
However in many cases a simple fan in a terminal unit such as a [ph::PhEntity.vav]
is more easily modeled as just a [ph::PhEntity.point].

## Fan Points
Standardized points for motors are extended to fans as follows:

  - [ph.points::FanEnableCmd]: command that permits or prohibits a fan to operate
  - [ph.points::FanRunCmd]: commands a fan to run or stop running
  - [ph.points::FanRunSensor]: senses the on/off state of a fan
  - [ph.points::FanSpeedModulatingCmd]: command for modulating fan speed

Modulating speed is defined as a percentage where 0% is off and 100% is full speed.

Permission from [ph.points::FanEnableCmd] may be required for [ph.points::FanRunCmd] to
take effect.

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

