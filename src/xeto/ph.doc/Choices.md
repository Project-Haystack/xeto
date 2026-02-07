<!--
title:      Choices
author:     Brian Frank
created:    17 Sep 2019
copyright:  Copyright (c) 2019, Project-Haystack
-->

# Overview
Choices define exclusive relationship between a set of marker tags.  For
example, [ph::DuctSection] is an exclusive choice - you choose one option
such as [ph::PhEntity.discharge] or [ph::PhEntity.return].  But you cannot use [ph::PhEntity.discharge]
and [ph::PhEntity.return] together.

Many common concepts in our ontology fit this pattern:
  - [ph::PointFunction]
  - [ph::CoolingProcess]
  - [ph::HeatingProcess]
  - [ph::DuctSection]
  - [pipeFluid]
  - [ph::PipeSection]

# Definition
Choices are defined as a subtype of [ph::PhEntity.choice] (which is itself a subtype
of [sys::Marker]).  They are registered on a given entity using [tagOn].  For
example, the [ph::PointFunction] choice identifies a point as [ph::PhEntity.sensor], [ph::PhEntity.cmd],
or [ph::PhEntity.sp].  The definition looks like this:

    def: ^pointFunction
    is: ^choice
    tagOn: ^point
    ---
    def: ^sensor
    is: ^pointFunction
    ---
    def: ^cmd
    is: ^pointFunction
    ---
    def: ^sp
    is: ^pointFunction

In the code above, pointFunction subtypes choice.  Then the exclusive
set of choices are subtypes of pointFunction.  This informs us that
when configuring a point, you should choose exactly one marker between
the three choices of sensor, cmd, and sp.

# Of Choices
There are some situations, where we wish the choice to reuse an existing
taxonomy tree where subtyping does not make sense.  For example, the
[pipeFluid] choice defines that we should annotate a [ph::PhEntity.pipe] with the type
of [ph::Fluid] it conveys.  In this situation, it would be awkward to have fluid
subtype pipeFluid.  Instead we can define the choice to use an arbitrary
set of subtypes using the [ph::PhEntity.of] tag.  Here is what the definition of pipeFluid
looks like:

    def: ^pipeFluid
    is: ^choice
    of: ^fluid
    tagOn: [^pipe, ^valve-actuator, ^pump-motor]

This definition informs us that when creating a pipe, valve, or pump that
we should select exactly one subtype of [ph::Fluid] to model the working fluid.

A given choice must use only one of these mechanisms.  If the choice defines
the `of` tag, then the referent defines the choice options (and the choice
must have no subtypes).  If the `of` tag is not defined, then it is assumed
that the choice subtypes define the options.