<!--
title:      Weather
author:     Brian Frank
created:    13 Apr 2021
copyright:  Copyright (c) 2021, Project-Haystack
-->

# Overview
Haystack standardizes the modeling of weather stations and their
associated observations.  The stations themselves are modeled using
the [ph::WeatherStation] spec.  The observations are modeled as
normalized [points](Points) along with the [ph::PhEntity.weather] marker tag.

# Stations
Weather stations model a geographic location where observations are
taken.  Stations might be an official government location; for example in
the US the NOAA provides an extensive database of station locations and their
associated data.  Many online weather services are virtual and approximate
data using several physical stations.  Or, it is possible to model site
local weather sensors using the Haystack weather model.

All weather stations must define the [ph::PhEntity.weatherStation] marker tag along
with the appropriate [ph::PhEntity.geoPlace] tags.  The [ph::PhEntity.tz] tag must also be defined.

Here is an example of a weather station entity:

    id: @richmond
    dis: "Weather for Richmond VA"
    weatherStation
    geoCoord: C(37.542979,-77.469092)
    geoCity: "Richmond"
    geoCountry: "US"
    geoState: "VA"
    tz: "New_York"

When a weather station is available for a site, the site should define
the [ph::PhEntity.weatherStationRef] tag to model the relationship.

# Points
All weather observation points must define the following tags:
  - [ph::PhEntity.weather]: marker to indicate entity is a weather point
  - [ph::PhEntity.weatherStationRef]: parent weather station associated with the point
  - [ph::PhEntity.sensor]: all observation points are considered sensor points

Additionally the standard point tags must be defined including [ph::PhEntity.point],
[ph::PhEntity.kind], [ph::PhEntity.unit], and [ph::PhEntity.enum].

The following are the standardized weather points that may be defined
for a given station:

  - [air temp](ph::AirTemp): dry bulb temperature in °C or °F
  - [ph::PhEntity.air] [ph::PhEntity.wetBulb]: web bulb temperature in °C or °F
  - [ph::PhEntity.air] [ph::PhEntity.feelsLike]: apparent temperature in °C or °F
  - [ph::PhEntity.air] [ph::PhEntity.dewPoint]: dew point temperature in °C or °F
  - [ph::PhEntity.air] [ph::PhEntity.humidity]: relative humidity in %RH
  - [ph::PhEntity.air] [ph::PhEntity.enthalpy]: total heat content in kJ/kg or BTU/lb
  - [atmospheric pressure](ph::AtmosphericPressure): barometric pressure in
    millibar or inHg
  - [ph::PhEntity.cloudage]: cloudiness as percentage
  - [ph::PhEntity.daytime]: boolean point that is true between sunrise and sunset
  - [ph::PhEntity.precipitation]: fallen atmospheric water vapor in mm or in
  - [solar irradiance](ph::SolarIrradiance): energy received from sun in W/m²
  - [ph::PhEntity.weatherCond]: enumeration for clear, cloudy, rain, snow, etc
  - [wind speed](ph::WindSpeed): wind speed in km/h or mph
  - [wind direction](ph::WindDirection): direction that wind originates in degrees
  - [ph::PhEntity.visibility]: distance at which light can be discerned in km or mile

Example of a weather observation point:

    // air temperature with a current value
    id: @richmond.temp
    dis: "Temperature for Richmond VA"
    weather
    air
    temp
    sensor
    point
    unit: "°F"
    kind: "Number"
    tz: "New_York"
    weatherStationRef: @richmond
    cur
    curVal: 56°F
    curStatus: "ok"

