# Overview

Specs which inherit from `sys::Enum` define a closed enumerated range of
string values.  Enum specs are effectively sealed and cannot be extended.

# Syntax

Enums are declared just like a dict spec with the enumerated range defined
as slots between curly braces:

```xeto
Suit: Enum {
  clubs
  diamonds
  hearts
  spades
}
```

The slot names define the range of valid string values for the enumerated type.
The slots must not define a type - they are implied to be items for the parent
enum type. The enum above specifies any value typed as Suit must be one of the
following string values: "clubs", "diamonds", "hearts", or "spades".

In the case when the range of string values are not valid slot names, then
define the string value via 'key' in the slot metadata:

```xeto
Suit: Enum {
  clubs    <key:"Clubs">
  diamonds <key:"Diamonds">
  hearts   <key:"Hearts">
  spades   <key:"Spades">
}
```

The enum spec above has a valid range of "Clubs", "Diamonds", "Hearts",
or "Spades".  Note in this case the slot names such as "clubs" is **not**
a valid string value for the enum.

You can also add arbitrary metadata to the enum and slots:

```xeto
Suit: Enum <icon:"playing-card"> {
  clubs    <key:"Clubs",    color:"black">
  diamonds <key:"Diamonds", color:"red">
  hearts   <key:"Hearts",   color:"red">
  spades   <key:"Spades",   color:"black">
}
```



