<!--
title:      PointPatterns
author:     Rick Jennings
created:    3 Jul 2026
copyright:  Copyright (c) 2026, Project-Haystack
-->

# Overview
The `ph.points` lib defines standardized Xeto specs for the points commonly
found on equipment.  This chapter describes the patterns these specs
follow so that points are modeled consistently across equipment and
can be queried by their Xeto types.

# Run and Enable
The on/off state of equipment is modeled with [ph.points::RunPoint]:

  - [ph.points::RunCmd]: commands equipment to run or stop running
  - [ph.points::RunSensor]: senses the actual on/off state of equipment

[ph.points::EnablePoint] models a secondary interlock used in
conjunction with a run point:

  - [ph.points::EnableCmd]: command that permits or prohibits
    equipment from running

Permission from an enable command may be required for a run command to
take effect.

# Controlling Equipment Capacity
Equipment with more than simple on/off control varies its capacity
either in discrete stages or by continuous modulation.  The two may
be combined when the capacity of individual stages is itself
modulated.

## Staged Capacity
Equipment with multiple discrete levels of capacity numbers its stages
sequentially as integers 1, 2, 3, etc. using the [ph::PhEntity.stage]
tag.  Each stage is commanded and sensed with its own run points, with
the stage tag identifying the stage.  For example, a two-stage heating
coil is commanded with two [ph.points::HeatRunCmd] points, one with
stage 1 and one with stage 2.  Each stage's actual on/off state is
proven with a matching [ph.points::HeatRunSensor].

## Modulating Capacity
[ph.points::ModulatingPoint] models the variable capacity of
equipment from 0% to 100%, such as valve position or motor speed:

  - [ph.points::ModulatingCmd]: commands equipment to operate at a
    percentage of capacity
  - [ph.points::ModulatingSensor]: senses the capacity at which
    equipment is operating

# Command and Sensor Pairs
A command models the desired state of equipment, while a sensor models
its actual state.  When equipment provides feedback, both should be
modeled as separate points.  For example a fan is commanded on/off with
[ph.points::FanRunCmd] and its actual status is proven with
[ph.points::FanRunSensor].  Likewise a damper is commanded to a
position with [ph.points::DamperCmd] and its actual position is sensed
with [ph.points::DamperSensor].  Comparing the pair detects failures
such as a fan commanded on that is not running.

# Setpoints
A setpoint models a value that guides control, either as a
limit that constrains a value or as the target that control drives a
value toward.  The same value may combine these roles, such as a
VAV whose airflow has a target setpoint along with minimum and
maximum airflow setpoints.

# Heating and Cooling
The run, enable, and modulating patterns are specialized for heating
and cooling equipment with the following specs:

  - [ph.points::HeatRunCmd], [ph.points::CoolRunCmd]: command heating
    or cooling equipment to run
  - [ph.points::HeatRunSensor], [ph.points::CoolRunSensor]: sense the
    on/off state of heating or cooling equipment
  - [ph.points::HeatEnableCmd], [ph.points::CoolEnableCmd]: permit or
    prohibit heating or cooling equipment from running
  - [ph.points::HeatModulatingCmd], [ph.points::CoolModulatingCmd]:
    modulate heating or cooling capacity
  - [ph.points::HeatModulatingSensor], [ph.points::CoolModulatingSensor]:
    sense heating or cooling capacity

# Duplicate Points
An entity may have several points that would otherwise be modeled
identically.  Such points should be differentiated with one of the following
mechanisms:

  - a slot value on the same spec, when points form a numbered or
    enumerated series, such as the [ph::PhEntity.stage] of a run
    command
  - a more specific spec, when points differ in scope, such as
    [ph.points::ZoneAirTempOccHeatingSp] versus
    [ph.points::ZoneAirTempUnoccHeatingSp]
  - a [ph::PhEntity.dis] tag with a human display name, when points
    have no modeled distinction, such as redundant sensors of the
    same quantity

# Examples

## Basic Equip Operation
This example combines the patterns above: an enable command
interlocked with a run command per the run and enable pattern,
and the run command paired with a run sensor to prove the actual
on/off state of the equipment.  The generic point specs are used
directly for any equip with simple on/off control.  Here are the
points for basic equip operation:

    // equip enable command
    id: @atlanta.equip.enable.cmd
    enable
    cmd
    point
    siteRef: @atlanta
    equipRef: @equip
    kind: "Bool"
    tz: "New_York"
    spec: @ph.points::EnableCmd

    // equip run command
    id: @atlanta.equip.run.cmd
    run
    cmd
    point
    siteRef: @atlanta
    equipRef: @equip
    kind: "Bool"
    tz: "New_York"
    spec: @ph.points::RunCmd

    // equip run sensor
    id: @atlanta.equip.run.sensor
    run
    sensor
    point
    siteRef: @atlanta
    equipRef: @equip
    kind: "Bool"
    tz: "New_York"
    spec: @ph.points::RunSensor

## Staged Heating
This example combines the patterns above: the generic run and
enable points specialized for heating, with each stage
differentiated by its [ph::PhEntity.stage] tag per the duplicate
points guidance.  The same point specs apply to any equip with
staged heating, such as an AHU or a hot water boiler.  Here are
the points for two heating stages:

    // stage 1 heat enable command
    id: @atlanta.boiler.stage1.heat.enable.cmd
    heat
    enable
    cmd
    stage: 1
    point
    siteRef: @atlanta
    equipRef: @boiler
    kind: "Bool"
    tz: "New_York"
    spec: @ph.points::HeatEnableCmd

    // stage 1 heat run command
    id: @atlanta.boiler.stage1.heat.run.cmd
    heat
    run
    cmd
    stage: 1
    point
    siteRef: @atlanta
    equipRef: @boiler
    kind: "Bool"
    tz: "New_York"
    spec: @ph.points::HeatRunCmd

    // stage 2 heat enable command
    id: @atlanta.boiler.stage2.heat.enable.cmd
    heat
    enable
    cmd
    stage: 2
    point
    siteRef: @atlanta
    equipRef: @boiler
    kind: "Bool"
    tz: "New_York"
    spec: @ph.points::HeatEnableCmd

    // stage 2 heat run command
    id: @atlanta.boiler.stage2.heat.run.cmd
    heat
    run
    cmd
    stage: 2
    point
    siteRef: @atlanta
    equipRef: @boiler
    kind: "Bool"
    tz: "New_York"
    spec: @ph.points::HeatRunCmd

## Modulated Cooling
This example combines the patterns above: the generic enable and
modulating points specialized for cooling, and the modulating
command paired with a modulating sensor to prove the capacity at
which equipment is operating.  Unlike staged heating, capacity
varies continuously from 0% to 100%, so a single set of points
models it without a stage tag.  The same point specs apply to any
equip with modulated cooling, such as an AHU or a chiller.  Here
are the points for modulated cooling:

    // cool enable command
    id: @atlanta.ahu.cool.enable.cmd
    cool
    enable
    cmd
    point
    siteRef: @atlanta
    equipRef: @ahu
    kind: "Bool"
    tz: "New_York"
    spec: @ph.points::CoolEnableCmd

    // cool modulating command
    id: @atlanta.ahu.cool.modulating.cmd
    cool
    modulating
    cmd
    point
    siteRef: @atlanta
    equipRef: @ahu
    kind: "Number"
    unit: "%"
    tz: "New_York"
    spec: @ph.points::CoolModulatingCmd

    // cool modulating sensor
    id: @atlanta.ahu.cool.modulating.sensor
    cool
    modulating
    sensor
    point
    siteRef: @atlanta
    equipRef: @ahu
    kind: "Number"
    unit: "%"
    tz: "New_York"
    spec: @ph.points::CoolModulatingSensor

# Equipment Points
These patterns are extended to specific equipment with specialized
point specs, for example [ph.points::MotorRunCmd],
[ph.points::FanRunSensor], and [ph.points::FanSpeedModulatingCmd].  See
the equipment chapters for the standardized points of each vertical:

  - [Motors](Motors): fans, pumps, and other motors
  - [AHUs](AHUs): air handling units
  - [VAVs](VAVs): variable air volume terminal units
  - [Zones](Zones): zone occupancy, hvac, air quality, and lighting
