---
layout : doc
title : Data Types
group : doc
comments : true
---
### Case Classes
Naturally, you represent your entities with them.

### All Standard Primitives
`Boolean`, `Byte`, `Char`, `Double`, `Float`, `Int`, `Long`, `Short`

### String
Strings support has a wrapped logic: if a `String` property is specified as part of some kind of key (unique, index) on a relational side it is represented as a `VARCHAR` and thus is limited to have a maximum length of only 256 characters, otherwise it is represented as by a `MEDIUMBLOB` which is limited to ~16MB of data.

### Option
Supported

### Collections
General types of immutable collections: `Seq`, `Set` and `Map` - are all supported. Please note that you should always use these general types instead of specific ones, i.e. `Seq` instead of `List`, `IndexedSeq` or `Vector` and etc.

### Ranges
A simple range of form `1 to 21` is supported. The "until"-ranges or the ones specifying a step are unsupported.

### Tuples
Supported

### Enumerations
Standard Scala enumerations are supported

### Joda Time
* `org.joda.time.DateTime` - to represent any given point in time of current era including timezone information with precision to milliseconds
* `org.joda.time.LocalDate` - to represent any date of current era without a timezone
* `org.joda.time.LocalTime` - to represent a time without a timezone
