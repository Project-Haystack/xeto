<!--
title:      Changes3to4
author:     Brian Frank
created:    23 Mar 2020
copyright:  Copyright (c) 2019, Project-Haystack
-->

# Overview
Haystack 3 defined a total of 230 tags. Of these, 180 remain unchanged.
The others have been modified as follows.

These are renamed or turned into conjuncts:

  - ahuRef => [ph::PhEntity.airRef]
  - airCooled => [ph::PhEntity.airCooling]
  - chilledWaterCool => [ph::PhEntity.chilledWaterCooling]
  - chilledWaterPlant => [ph::ChilledWaterPlant]
  - chilledWaterPlantRef => [ph::PhEntity.chilledWaterRef]
  - closedLoop => [ph::PhEntity.condenserClosedLoop]
  - co => [ph::PhEntity.co] [ph::PhEntity.concentration]
  - co2 => [ph::PhEntity.co2] [ph::PhEntity.concentration]
  - constantVolume  => [ph::PhEntity.constantAirVolume]
  - coolOnly => [ph::PhEntity.coolingOnly]
  - dew => [ph::PhEntity.dewPoint]
  - dxCool => [ph::PhEntity.dxCooling]
  - elecHeat => [ph::PhEntity.elecHeating]
  - elecMeterLoad => [ph::PhEntity.elecRef]
  - elecMeterRef => [ph::PhEntity.elecRef]
  - elecPanel => [ph::ElecPanel]
  - elecPanelOf => [ph::PhEntity.elecRef]
  - elecReheat => [ph::PhEntity.reheat] [ph::PhEntity.elecHeating]
  - gasHeat => [ph::PhEntity.naturalGasHeating]
  - gasMeterLoad => [ph::PhEntity.naturalGasRef]
  - hisInterpolate => [ph::PhEntity.hisMode]
  - hotWaterHeat => [ph::PhEntity.hotWaterHeating]
  - hotWaterPlant => [ph::HotWaterPlant]
  - hotWaterPlantRef => [ph::PhEntity.hotWaterRef]
  - hotWaterReheat => [ph::PhEntity.hotWaterHeating]
  - lights => [ph::LightLevel]
  - lightLevel => [ph::PhEntity.illuminance]
  - lightsGroup => [ph::LightingZoneSpace] or [ph::PhEntity.luminaire]
  - mag => [ph::PhEntity.magnitude]
  - max => [ph::PhEntity.maxVal]
  - min => [ph::PhEntity.minVal]
  - occupancyIndicator => [ph::PhEntity.occupancy]
  - oil => [ph::PhEntity.fuelOil]
  - openLoop => [ph::PhEntity.condenserOpenLoop]
  - rooftop => [ph::PhEntity.rtu]
  - screw => [ph::PhEntity.rotaryScrew]
  - sitePanel => [ph::PhEntity.elecRef]
  - steamHeat => [ph::PhEntity.steamHeating]
  - steamMeterLoad => [ph::PhEntity.steamRef]
  - steamPlant => [ph::SteamPlant]
  - steamPlantRef => [ph::PhEntity.steamRef]
  - subPanelOf => [ph::PhEntity.elecRef]
  - sunrise => [ph::PhEntity.daytime]
  - uv => [ph::PhEntity.unitVent]
  - variableVolume => [ph::PhEntity.variableAirVolume]
  - vavMode => [ph::PhEntity.hvacMode]
  - waterCooled => [ph::PhEntity.waterCooling]
  - waterMeterLoad => `xxxWaterRef` such as [ph::PhEntity.chilledWaterRef]
  - weather => [see weather](#weather)
  - weatherRef => [see weather](#weather)
  - weatherPoint => [see weather](#weather)

These tags are not ported:

  - connection => [see networks](#networks)
  - device1Ref => [see networks](#networks)
  - device2Ref => [see networks](#networks)
  - reheating  => was only used by one VAV max flow sp

# Weather
In Haystack 3, weather stations were modeled via the `weather` tag, and
their points were modeled via the `weatherPoint` tag.  In Haystack 4, we
use the [ph::PhEntity.weather] marker on the points as the [pointSubject].  And stations
are modeled via the [ph::PhEntity.weatherStation] tag.  Likewise, the `weatherRef` tag
is now renamed to `weatherStationRef`.

Weather points are mapped from Haystack 3 to Haystack 4 as follows:

    Haystack 3            Haystack 4             Notes
    -------------------   --------------------   ------------------------
    weatherCond           weatherCond            same
    air temp              air temp               same
    wetBulb temp          air wetBulb            pair with air not temp
    apparent temp         air feelsLike          renamed, pair with air not temp
    dew temp              air dewPoint           pair with air not temp
    humidity              air humidity           pair with air
    barometric pressure   atmospheric pressure   renamed
    sunrise               daytime                renamed
    precipitation         precipitation          same
    cloudage              cloudage               same
    solar irradiance      solar irradiance       same
    wind direction        wind direction         same
    wind speed            wind speed             same
    visibility            visibility             same

See [ph::PhEntity.weatherStation] children section for full list of points.

# Lighting
Most tags related to lighting have not been included pending the work
under [WG 705](https://project-haystack.org/forum/topic/705)

# Networks
Haystack 3 included a prototype for modeling networks, devices, and
connections between them.  It was never implemented by any vendors and
was not complete, so it has not been ported to Haystack 4.  However, some
of the core concepts are found in the `phIct` library including:
  - [ph::PhEntity.network]
  - [ph::PhEntity.device]
  - [ph::PhEntity.protocol]

# Symbols
Symbol is added as a new scalar type in Haystack 4.  In Zinc, it is represented
as `^name` and in JSON with the "y:" prefix.  The format version was left
as "3.0" to minimize ecosystem disruption since symbols are only used with
defs and unlikely to occur in normal Haystack data.


