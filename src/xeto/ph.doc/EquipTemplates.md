<!--
title:      EquipTemplates
author:     Brian Frank
created:    5 Oct 2026
copyright:  Copyright (c) 2026, Project-Haystack
license:    Licensed under the Academic Free License version 3.0
-->

# Overview
An *equip template* is an [ph::Equip] spec which models one specific
vendor product: a VAV controller, a power meter, a drive.  The template
declares every point the product exposes, the Haystack type of each
point, and the protocol address used to read or write it over BACnet
and/or Modbus.

A template is written once per product and shared as a Xeto lib.  Tools
then use it to:

  - instantiate the equip and its points as recs
  - bind each point to a connector using its protocol address map
  - validate deployed recs against the template

Templates are built from these standard libs:

  - [ph.points](ph.points::index), [ph.points.sugar](ph.points.sugar::index),
    and [ph.elec](ph.elec::index): the Haystack point types
  - [ph.attrs](ph.attrs::index): nameplate attributes
  - [ph.protocols](ph.protocols::index): the protocol address specs

# Lib Layout
Put each product, or closely related family of products, in its own lib.
If it's a community contribution, then use the `cc.` prefix.

```xeto
pragma: Lib <
  doc: "Acme VAV-100 terminal unit controller"
  version: "0.1.0"
  depends: {
    { lib: "sys" }
    { lib: "sys.files" }     // PdfFile for resources
    { lib: "ph" }
    { lib: "ph.attrs" }
    { lib: "ph.points" }
    { lib: "ph.points.sugar" }
    { lib: "ph.protocols" }
  }
>
```

Add `ph.elec` for electrical points.  Device configuration the template
depends on goes in the lib's `readme.md` - see [Configuration](#configuration).

# Equip Spec
Subtype the most specific Haystack equip type which describes the product,
such as [ph::Vav], [ph::AcElecMeter], or [ph::ZoneMonitor].  The template then
inherits the ontology markers of that type and fits the standard queries
and rules written against it.  When no specific type fits, subtype [ph::Equip]
directly.

```xeto
// Acme VAV-100 BACnet/Modbus terminal unit controller.
//
// Source: VAV-100 Integration Guide rev 3, Table 5 "Points List"
AcmeVav100 : Vav {
  attrs: Query {
    ManufacturerAttr { val: "Acme" }
    ModelSeriesAttr { val: "VAV-100" }
    DimensionHeightAttr { val: 120mm }
  }
  resources: List? <of:File> {
    PdfFile {
      dis: "VAV-100 Integration Guide"
      uri: "https://acme.example/docs/vav100.pdf"
    }
  }
  points: {
    ...
  }
}
```

Conventions:

  - **doc comment**: cite the vendor documents and section the point
    list was transcribed from
  - **attrs**: nameplate data as an `attrs: Query` of `ph.attrs` specs
    with a `val` - see [Attrs](#attrs)
  - **resources**: manuals and datasheets as `sys.files` specs
  - **points**: one named slot per point the product exposes

## Attrs
Include the universal attrs from `ph.attrs` whenever the vendor
documents them, since they hold for every unit of the product:

  - [ph.attrs::ManufacturerAttr]
  - [ph.attrs::ModelSeriesAttr]
  - [ph.attrs::ModelNumberAttr] when the template is for one model
  - [ph.attrs::DimensionLengthAttr], [ph.attrs::DimensionWidthAttr],
    [ph.attrs::DimensionHeightAttr]
  - [ph.attrs::WeightAttr]

Never put per-unit attrs such as [ph.attrs::SerialNumberAttr] or
[ph.attrs::ManufactureDateAttr] in a template.  The other domain
attrs (ratings, temperatures, electrical) are not yet fully defined
by the Haystack ontology; omit them unless specifically required.

## Variants
When the product exposes different points depending on how it is
configured, define an abstract base with the common points and one
concrete subtype per configuration.  For example, if an electric meter
changes its voltage points with its wiring mode:

```xeto
AcmeElecAcMeter : AcElecMeter <abstract> {
  points: { ... common points ... }
}

AcmeElecAcLLMeter : AcmeElecAcMeter {
  points: { ... line-to-line only points ... }
}
```

The subtype's points merge with its inherited points.  Only concrete
templates may be instantiated.

# Points
Each point is a named slot in the `points` query:

```xeto
points: {
  // Zone temperature from the wall sensor
  ai1: ZoneAirTempSensor? {
    dis: "Zone Temp"
    unit: "°F"
    bacnetCurAddr: { addr: "AI1" }
    modbusCurAddr: { addr: "300001", encoding: "s2", scale: "*0.1" }
  }
}
```

## Slot Names
Point slots must be **named**, never auto-named.  The slot qname such as
`acme.vav100::AcmeVav100.points.ai1` is the identity of the point: a
deployed rec stores it in its `spec` tag, and connectors may store it as
the point's address.  So:

  - derive the name from the vendor address or register, such as `ai1`,
    `av34`, `msv2`, or `reg16384`
  - keep names stable across versions: renaming a slot orphans every rec
    deployed from it
  - never reuse a removed name for a different point

## Optional Points
Declare points as maybe types with `?`.  A template describes what the
product *can* expose; most installations deploy only a subset.  Use a
required point only when the equip is meaningless without it.

## Point Types
Choose the most specific standard spec which describes the point's
semantics - [ph.points::ZoneAirTempSensor], [ph.elec::ElecAcFreqSensor],
and so on.  Narrow it further with tags in the slot body when the spec
leaves a dimension open, such as `phase: "L1"` on an electrical sensor.
See [Points](Points.md), [PointPatterns](PointPatterns.md), and the
vertical chapters such as [Meters](Meters.md) for the rules.

Many points of a template may share one type.  Validation anchors each
deployed rec to its slot by its `spec` tag, so points which differ only
by name still match exactly one constraint each - see
[Query Constraints](doc.xeto::Constraints#query-constraints).

When no standard spec captures the point's semantics - vendor
configuration, diagnostics, values Haystack does not model yet - use a
generic point type with the `notHaystack` meta marker and the point's
function marker:

```xeto
// Time to hold bypass occupancy before reverting
av34 : NumberPoint? <notHaystack> {
  sp
  dis: "Bypass Time"
  unit: "min"
  bacnetCurAddr: { addr: "AV34" }
  bacnetWriteAddr: { addr: "AV34" }
}
```

The generic types are [ph::NumberPoint] (requires a `unit`; use
`unitless` for codes and other dimensionless values),
[ph::BoolPoint], and [ph::EnumPoint].  Always add exactly one of
`sensor`, `sp`, or `cmd`.  Never misuse a standard type which is close but
semantically wrong: a wrong standard type is worse than `notHaystack`.

## Point Defaults
Values in the slot body are defaults applied when the point is
instantiated:

  - `dis`: the vendor's point name
  - `unit`: the unit the value is reported in *after* scaling; when the
    device's unit is configurable, use the factory default and list the
    setting as a [configuration](#configuration) requirement
  - `enum`: the enum spec of an `EnumPoint`, or `"falseText,trueText"`
    on a `BoolPoint`
  - tags which narrow the type such as `phase`

## Enums
Model multi-state points with a lib local [sys::Enum] named with the
vendor prefix, and reference it from the point:

```xeto
// Table 6-15: channel 1 protocol
AcmeCommProtocolEnum : Enum {
  modbus, bacnet
}

msv5 : EnumPoint? <notHaystack> {
  sp
  enum: Obj <val:AcmeCommProtocolEnum>
  bacnetCurAddr: { addr: "MSV5" }
}
```

Enum items are in state order.  When the vendor documents contradict
each other, note which reading was chosen and why in the enum's doc.

# Protocol Addresses
Addresses are authored with the global slots defined by
[ph.protocols](ph.protocols::index).  Each global is named by protocol
and function:

| Global              | Type                         | Function
|---------------------|------------------------------|--------------------------
| `bacnetCurAddr`     | [ph.protocols::BacnetAddr]   | read the current value
| `bacnetWriteAddr`   | [ph.protocols::BacnetAddr]   | write a writable point
| `bacnetHisAddr`     | [ph.protocols::BacnetAddr]   | trend log to sync history
| `modbusCurAddr`     | [ph.protocols::ModbusAddr]   | read the current value
| `modbusWriteAddr`   | [ph.protocols::ModbusAddr]   | write a writable point

The function is decided by which global carries the address.  A point
may author any combination: a read-only sensor has only a cur address; a
writable setpoint has both cur and write, usually to the same object or
register.  A point may author both BACnet and Modbus addresses when the
product speaks both.

## BACnet
A [ph.protocols::BacnetAddr] is an object type code plus the object
instance number, such as `AI3`, `AV34`, `BO2`, or `MSV5`.  See
[ph.protocols::BacnetAddr] for the table of codes.

```xeto
sp1 : ZoneAirTempSp? {
  bacnetCurAddr:   { addr: "AV10" }
  bacnetWriteAddr: { addr: "AV10" }
  bacnetHisAddr:   { addr: "TL3" }
}
```

Only objects with a priority array (outputs and most values) are
writable.  Author `bacnetHisAddr` only when the device has a trend log
for the point.

## Modbus
A [ph.protocols::ModbusAddr] carries everything needed to decode the
register:

| Field       | Description
|-------------|-----------------------------------------------------------
| `addr`      | 6-digit extended Modicon address (required)
| `encoding`  | `bit`, `u1`, `u2`, `u4`, `s1`, `s2`, `s4`, `s8`, `f4`, `f8` (required)
| `bitIndex`  | bit 0-15 within a register for `bit` encoding
| `byteOrder` | byte/word order for numeric encodings: `be` (default), `le`, `leb`, `lew`
| `scale`     | op chain applied to the raw value
| `access`    | `r`, `rw`, or `w` - informational only
| `dis`       | vendor's register name

Byte order: `leb` swaps the bytes within each 16-bit register so it
applies to single register encodings too; `lew` swaps register order so
it only matters for 32 and 64-bit encodings; `le` does both.  Byte order
does not apply to `bit` encoding.

Read versus write is decided by which global carries the address, never
by `access`: an address in `modbusWriteAddr` is written whatever its
`access` says.  Author `access` only to record the vendor's documented
register access.

The address must be the 6-digit form where the leading digit selects
the register type and the remaining 5 digits are the 1-based register
number:

    0xxxxx  Coil              000001-065536
    1xxxxx  Discrete Input    100001-165536
    3xxxxx  Input Register    300001-365536
    4xxxxx  Holding Register  400001-465536

Vendor documents often list 0-based register offsets, hex, or 5-digit
addresses.  Convert carefully: holding register offset `16384` (`0x4000`)
is address `416385`.

The `scale` expression is a chain of `[op] [number]` terms applied left
to right with no precedence; every term needs its operator.  So
`"+32768 /10"` means `(raw + 32768) / 10`.  Use it to convert the raw
value into the point's `unit`: a register reporting watts on a point in
kW uses `"*0.001"`.  There is no unit field on the address; the point's
own `unit` applies.  On writes the scale is applied in reverse, so a
`"*0.1"` register written with 72.5 receives the raw value 725.

```xeto
reg16418 : ElecAcTotalNetActivePowerSensor? {
  dis: "Total System Power"
  unit: "kW"
  modbusCurAddr: { addr: "416419", encoding: "f4", scale: "*0.001" }
}

do0 : BoolPoint? <notHaystack> {
  cmd
  modbusCurAddr:   { addr: "400101", encoding: "bit", bitIndex: 0 }
  modbusWriteAddr: { addr: "400101", encoding: "bit", bitIndex: 0 }
}
```

# Configuration
A template is only semantically correct when the device is configured
the way the template assumes: sign conventions, primary versus
secondary values, wiring modes, units.  Document every such requirement
in the lib's `readme.md` under a "Configuration requirements" heading,
citing the setting name and manual section:

```md
## Meter configuration requirements

 * "VAR/PF Convention" must be set to "IEC" (User Manual Section 4.8.4)
 * Analog measurements must be read in "Primary Mode" (Section 6.3.7)
```

When a setting selects between variants, say which concrete template
matches each value.

# Validation
Recs deployed from a template carry the point slot qname in their `spec`
tag.  Validating the equip with graph validation checks its points
against the template's `points` query: each anchored rec is checked
against its own slot, and any required point without a rec is reported.

