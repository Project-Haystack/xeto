# Grammar

The formal BNF grammar for Xeto:


```
<libFile>      :=  [<namedSpec> | <namedData>]*
<dataFile>     :=  <data> | <namedData>*  // single scalar/dict or list of named dicts
<namedSpec>    :=  <name> ":" <spec> <nl>
<namedData>    :=  <ref> ":" <dict> <nl>

<spec>         :=  [<type> [<meta>]] <specBody> // must have at one
<specBody>     :=  <specSlots> | <specVal>
<specVal>      :=  <scalar>
<specSlots>    :=  "{" [<specSlot> <endOfObj>]* "}"
<specSlot>     :=  [<leadingDoc>] ( <markerOnly> | <namedSpec> | <spec> | <embeddedMeta>) [<trailingDoc>]
<markerOnly>   :=  <markerName> [<meta>]
<endOfObj>     :=  ( [","] <nl> ) | ","
<meta>         :=  "<" <dictTags> ">"
<embeddedMeta> :=  "<" (<dictMarkerTag> | <dictMarkerTag>) ">"

<data>            :=  <dict> | <dataScalar> | <ref> | <spec>
<dataType>        :=  <typeSimple>    // may want to allow List<of>
<dataScalar>      :=  [<dataType>] <scalar>
<dict>            :=  [<dataType>] "{" <dictTags> "}"
<dictTags>        :=  [<dictTag> <endOfObj>]*
<dictTag>         :=  <dictMarkerTag> | <dictNamedTag> | <dictUnnamedTag> |
                      <dictIdTag> | <dictNamedIdTag>
<dictMarkerTag>   :=  <name>
<dictNamedTag>    :=  <name> ":" <data>
<dictUnnamedTag>  :=  <data>
<dictIdTag>       :=  <ref> ":" <dict>
<dictNamedIdTag>  :=  <name> <ref> ":" <dict>

<type>         :=  <typeMaybe> | <typeAnd> | <typeOr> | <typeSimple>
<typeMaybe>    :=  <typeSimple> "?"
<typeAnd>      :=  <typeSimple> ("&" <typeSimple>)+
<typeOr>       :=  <typeSimple> ("|" <typeSimple>)+
<typeSimple>   :=  <qname>

<leadingDoc>   := (<lineComment>)*
<trailingDoc>  := <lineComment>

<qname>        :=  [<dottedName> "::"] <dottedName>
<dottedName>   :=  <name> ("." <name>)*
<name>         :=  <alpha> <nameRest>*
<markerName>   :=  <alphaLower> <nameRest>*
<nameRest>     :=  alpha | digit | '_'
<ref>          :=  "@" <refChar>* <refEnd> [<sp> <quotedStr>]
<refChar>      :=  <alpha> | <digit> | "_" | "~" | ":" | "-"
<refEnd>       :=  <alpha> | <digit> | "_" | "~"  // cannot end with ":" or "-"
<alpha>        :=  alphaLower | alphaUpper
<alphaLower>   :=  'a' - 'z'
<alphaUpper>   :=  'A' - 'Z'
<digit>        :=  '0' - '9'
<sp>           :=  ' '

<scalar>       :=  see below
<quotedStr>    :=  single quoted string, see below
```

# Scalars

Scalar values may take one of the following formats:
  - single double-quoted string such as "hi"
  - triple double-quoted strings such as """my name is "Brian", hi!"""
  - heredocs bracketed with "---"
  - numbers with embedded units/symbols such as 123% or 2023-03-04
  - refs that start with "@" (see above)

Quoted strings use the same backslash escape sequence as C languages:
  - '\n'  for newline
  - '\\'  for backslash itself
  - '\"' for double-quote itself (triple quoted string does not require escaping)
  - '\u2023" unicode hex value

Triple quoted strings follow these indentation normalization rules:
  - If opening triple quote line is empty, then next line is the first line
  - Indentation is based on left-most line/closing quote
  - Any leading spaces are trimmed based on inferred indentation

Heredocs use three or more "-" and are closed with an equal number of dashes.
The work just like triple quoted strings regarding indentation normalization
rules. But heredocs ignore backslash escape sequences.

Number literals must start with an ASCII digit or "-" followed by an ASCII digit.
Any of the following characters are matched to tokenize the number literal:
  - ASCII digit or letter 0-9, A-Z, a-z
  - "." dot (0x2E)
  - "-" dash (0x2D)
  - ":" colon (0x3A)
  - "/" forward slash (0x2F)
  - "$" dollar sign (0x24)
  - "%" percent sign (0x25)
  - any Unicode character > 0x7F

# Legend
Legend for BNF Grammar:

```
  :=      is defined as
  <x>     non-terminal
  "x"     literal
  'x'     char literal
  [x]     optional
  (x)     grouping
  s-e     inclusive char range
  x*      zero or more times
  x+      one or more times
  x|x     or
```

