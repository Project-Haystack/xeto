<!--
title:      TimeZones
author:     Brian Frank
created:    22 Feb 2011
copyright:  Copyright (c) 2015, Project-Haystack
license:    Licensed under the Academic Free License version 3.0
-->

# Overview
Xeto requires all DateTimes to carry their actual timezone versus just
a UTC offset.  This comes from its heritage modeling time-series data that is
the foundation for sensor and operational data.  This makes it critical to define
a proper, unambiguous representation of time and timezones.  Timezones are
identified using a city name from the [zoneinfo database](http://en.wikipedia.org/wiki/Tz_database)
(discussed below) and standardized by the [sys::TimeZone] enum.

# Concepts
We use the term timezone to encapsulate two concepts: offset
from UTC and daylight saving time rules. For example, US Eastern Standard
Time is -5hrs from UTC. But between 2am on the second Sunday of March and 2am
on the first Sunday in November is daylight savings time (DST) and is -4hrs
from UTC.

Because timezones are such a critical aspect of time-series data, Xeto
requires a comprehensive model and database of timezones. Timezones are quite
problematic for computers because they are a political abstraction versus a
scientific abstraction. This means that a given region might change its timezone
rules (either UTC offset or DST rules) over time. For example, in 2007 the US
changed the dates for when DST starts and ends. This means that computing a date
in 2000 uses different rules than 2010 (we call these historical timezones).

# Zoneinfo
Luckily, there is a database that keeps track of these rules across regions
and time. Xeto uses the zoneinfo database, which is also used by Unix and
many programming languages.  The zoneinfo database uses a convention of naming
timezones as "Continent/City". For example, US Eastern time is canonically
identified as "America/New_York". Since there are no duplicate city names
between continents, the city name also works as a canonical identifier.

All timezones are represented using the city name of an entry
in the zoneinfo database.

# Common TimeZones
Here are some common timezones used in the United States:
- `New_York`: US Eastern Time
- `Chicago`: US Central Time
- `Denver`: US Mountain Time
- `Los_Angeles`: US Pacific Time
- `Phoenix`: Arizona Time (Mountain Time with no DST)

The list above is by no means comprehensive.  The full list of timezones
is defined by the [sys::TimeZone] enum; also see the
[zoneinfo website](https://www.iana.org/time-zones).

