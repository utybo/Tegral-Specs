# **DRAFT** Tegral Validate design document

> **Note**
> This is a draft for a potential new module. If anyone sees this, feel free to discuss the ideas presented here! No development has started yet

## Executive summary

Tegral Validate is a proposed library that would provide an easy-to-use API for performing validation against Kotlin values.

In a somewhat similar story to that of Koin and Tegral DI, Tegral Validate is inspired by Konform.

Here's an example of a potential API:

```kotlin
// ------ VALIDATING A SIMPLE OBJECT ------
data class Person(val name: String, val email: String) {
    init { isValidPerson.validate(this) }
}

val isValidPerson = createValidator<Person> {
    // Using reflection
    Person::name / isNotBlank()
    // Without reflection
    that(".email") { email } / isNotBlank()
}

// ------ VALIDATING A MORE COMPELEX OBJECT ------
data class Group(val people: List<Person>) {
    init { groupValidator.validate(this) }
}

val groupValidator = createValidator<Group> {
    Group::people / {
        Person::size / isIn(2..100)
        this / each(isValidPerson)
    }
}
```

## Validation

### `Validator<T>`

Tegral Validate (hereafter TV) is based on a single core type: a **validator** (similar to a _predicate_), which takes in a value and returns a **validation result**.

```kotlin
interface Validator<in T> {
    fun checkValid(input: T): ValidationResult
}

sealed class ValidationResult {
    object Valid

    class Invalid(reasons: List<...>) : ValidationResult()
}

// This class is subject to changes
data class ValidationFailureReason(val message: String, val constraint: String?)
```

Here's an example of a validator that implements a simple non-zero check for integers.

```kotlin
class NotEqualToZeroValidator : Validator<Int> {
    override fun checkValid(input: Int) {
        return if (input != 0) Valid
        else Invalid(
            listOf(
                ValidationFailureReason(
                    message = "Value was 0",
                    constraint = "Value must be different than 0"
                )
            )
        )
    }
}

fun main() {
    val validator = NotEqualToZeroValidator()
    checkValid(120) // Valid
    checkValue(456) // Valid
    checkValue(0) // Invalid(List(...))
}
```

### Core validators

The following core validators must be implemented and provided.

#### General validators

| Validator                           | Equivalent operation |
| ----------------------------------- | :------------------: |
| `EqualValidator(referenceValue)`    |         `==`         |
| `NotEqualValidator(referenceValue)` |         `!=`         |
| `SameValidator(referenceValue)`     |        `===`         |
| `NotSameValidator(referenceValue)`  |        `!==`         |
| `NullValidator`                     |      `== null`       |
| `NotNullValidator`                  |      `!= null`       |
| `IsValidator(referenceType)`        |  `is referenceType`  |
| `IsNotValidator(referenceType)`     | `is! referenceType`  |

#### `Comparable<T>` (including all numbers)

| Validator                     | Equivalent operation     |
| ----------------------------- | ------------------------ |
| `LessThanValidator`           | `candidate < reference`  |
| `LessThanOrEqualValidator`    | `candidate <= reference` |
| `GreaterThanValidator`        | `candidate > reference`  |
| `GreaterThanOrEqualValidator` | `candidate >= reference` |

#### Floating-point numbers (Float, Long)

All of the following validators are identical to their non-`WithPrecision` counterpart, except that they except a "tolerance" delta. For example, `candidate == reference` becomes `candidate in ((reference - tolerance)..(reference + tolerance)`.

The tolerance value **must** be `>= 0`.

- `EqualWithPrecisionValidator`: Same as its non-precision counterpart, but also uses a "tolerance" delta
- `NotEqualWithPrecisionValidator`: Same as its non-precision counterpart, but also uses a "tolerance" delta
- `LessThanWithPrecisionValidator`: Same as its non-precision counterpart, but also uses a "tolerance" delta
- `LessThanOrEqualWithPrecisionValidator`: Same as its non-precision counterpart, but also uses a "tolerance" delta
- `GreaterThanWithPrecisionValidator`: Same as its non-precision counterpart, but also uses a "tolerance" delta
- `GreaterThanOrEqualWithPrecisionValidator`: Same as its non-precision counterpart, but also uses a "tolerance" delta

### Combination validators

TV also provides the following built-in validators:

#### `AllValidValidator`

Is valid if all provided validators are also valid. Equivalent to a logical "AND" operation

Is also considered valid if provided with an empty validators list (matches Kotlin's `Collection.all`).

```kotlin
class AllValidValidator<in T>(val validators: List<Validator<T>>) : Validator<T> {
    // ...
}
```

#### `AnyValidValidator`

Is valid if any of the provided validators is also valid. Equivalent to a logical "OR" operation

Is considered invalid if provided with an empty validators list (matches Kotlin's `Collection.any`).

#### `NotValidValidator`

Is valid if the provided validator is invalid. This is equivalent to a logical "NOT" operation

#### `OnlyOneValidValidator`

Is valid if exactly one of the provided validators is valid. Equivalent to a logical "XOR" operation.

Is considered invalid if provided with an empty validators list.

#### `NotNullAndValidValidator`

Is valid if the value is not null AND the provided validator is valid.

This is useful if you want to check that a value of type `T?` is non-null (i.e. of type `T`) and also validate the type `T` with validators that may not support values of type `T?`.

#### `NullOrValidValidator`

Is valid if:

- the value is null
- OR if the value is not null and is valid against the provided validator

This is similar to Konform's `ifPresent`: you want to use a given validator only if the value is not null, and if it _is_ null it is considered valid.

#### `CustomConstraintValidator`

Is valid if the provided value is valid. Additionally, if invalid, replaces the original validator's `constraint` string by the provided constraint value.

To ease the creation of such validators, the following function is provided:

```kotlin
fun <T> Validator<T>.withConstraint(constraint: String): CustomConstraintValidator<T> {
    return CustomConstraintValidator(this, constraint)
}
```

### Collection validators

#### `SizeValidator`

Is valid if the provided validator is true for the size of this collection.

For example:

```kotlin
val validator = SizeValidator(MoreThanOrEqualValidator(3))
validator.checkValid(listOf(1, 2, 3)) // Valid
validator.checkValid(listOf(1, 2, 3, 4)) // Valid
validator.checkValid(listOf(1, 2)) // Invalid
```

#### `AllElementsValidator`

Is valid if the given validator considers all of the elements in the collection to be valid.

Is considered valid if the collection has no elements.

#### `AnyElementValidator`

Is valid if the given validator considers all of the elements in the collectino to be valid.

Is considered invalid if the collection has no elements.

#### `OnlyOneElementValidator`

Is valid if exactly one of the provided validators is valid.

Is considered invalid if the collection has no elements.

### Nested validators (`NodeValidator`)

In order to support values that contain other values, a special validator, `NodeValidator`, provides the concept of _mappers_.

A `NodeValidator<T>` is made of a list of _mapped validators_. Each mapped validators contains:

- A mapper (`Mapper<T, R>`), which maps the value of type `T` to a value of type `R` for further validation.
- A validator for `R`

Here is the signature for a mapper:

```kotlin
interface Mapper<in T, out R> {
    val valuePath: String
    fun get(input: T): R
}
```

The `valuePath` allows for creating useful debugging information when validating deeply nested objects.

### String validators

These validators are specifically designed for strings and character sequences.

#### `MatchesPatternValidator`

Is valid if the given character sequence matches the provided RegEx pattern.

#### `LengthValidator`

Identical to `SizeValidator` but for string lengths.

#### NotBlank, NullOrBlank, Blank, NotEmpty, NullOrEmpty, Empty

These validators are comparable to built-in Kotlin string predicates:

| Validator              | Equivalent Kotlin predicate |
| ---------------------- | --------------------------- |
| `NotBlankValidator`    | `isNotBlank()`              |
| `NullOrBlankValidator` | `isNullOrBlank()`           |
| `BlankValidator`       | `isBlank()`                 |
| `NotEmptyValidator`    | `isNotEmpty()`              |
| `NullOrEmptyValidator` | `isNullOrEmpty()`           |
| `EmptyValidator`       | `isEmpty()`                 |

### Misc. validators

#### `PredicateValidator`

Uses a predicate and a custom message and constraint to validate a value.

```kotlin
class PredicateValidator<in T>(
    val predicate: (T) -> Boolean,
    val message: String,
    val constraint: String
) : Validator<T> {
    // ...
}
```

#### `In(...)Validator`

Is valid if the value is "in" (in the `in` operator sense) another value. Said value can be a collection (e.g. `setOf(...)`), a range (e.g. `Range`, etc.)

There are several variants of this validator depending on the criteria:

- InRangeValidator
- InCollectionValidator
- Maybe others?

### Examples

Here are examples of manually-created validators.

#### Example 1: Konform's validateEvent

Here is an equivalent to Konform's `validateEvent` example.

##### Data type

```kotlin
data class Person(val name: String, val email: String?, val age: Int)

data class Event(
    val organizer: Person,
    val attendees: List<Person>,
    val ticketPrices: Map<String, Double?>
)
```

##### Validator

```kotlin
// Note: these validators are split for clarity. In a real ValidatorDSL
// use-case, this would probably be defined as a single validator using the DSL.

val organizerValidator = NodeValidator<Person>(
    listOf(
        mapperOf(".email") { email } to NotNullAndValidValidator(
            MatchesPatternValidator(".+@bigcorp.com")
                .withConstraint("Organizers must have a BigCorp email address")
        )
    )
)

val attendeeValidator = NodeValidator<Person>(
    listOf(
        mapperOf(".name") { name } to LengthValidator(MoreThanOrEqualValidator(2)),
        mapperOf(".age") { age } to MoreThanOrEqualValidator(18)
            .withConstraint("Attendees must be 18 years or older"),
        mapperOf(".email") { email } to NullOrValidValidator(
            MatchesPatternValidator(".+@.+\\..+")
                .withConstraint("Please provide a valid email address")
        )
    )
)

val ticketPriceValidator = NodeValidator<Entriy<String, Double?>>(
    listOf(
        mapperOf(".value") { value } to NullOrValidValidator(
            MoreThanOrEqual(0.01)
        )
    )
)

val eventValidator = NodeValidator<Event>(
    listOf(
        mapperOf(".organizer") { organizer } to organizerValidator,
        mapperOf(".attendees") { attendees } to AllValidator(
            listOf(
                SizeValidator(LessThanOrEqualValidator(100)),
                AllElementsValidator(attendeeValidator)
            )
        ),
        mapperOf(".ticketPrices") { ticketPrices } to AllValidator(
            SizeValidator(MoreThanOrEqualValidator(1)),
            AllElementsValidator(ticketPriceValidator)
        ),

    )
)
```

#### Example 2: Kalidation's README example

```kotlin
// Initial plans for TV do not include a built-in e-mail validator
val emailValidator = MatchesPatternValidator(/* some e-mail regex */)

val fooValidator = NodeValidator<Foo>(
    mapperOf(".bar") { bar } to AllValidValidator(
        NotBlankValidator(),
        InCollectionValidator(setOf("GREEN", "WHITE", "RED"))
    ),
    mapperOf(".bax") { bax } to AllValidValidator(
        SizeValidator(MoreThanOrEqualValidator(3)),
        emailValidator
    ),
    // No available or planned alternative for Foo::baz { validByScript(...) }
    mapperOf() { this } to PredicateValidator(predicate = { it.validate() }, constraint = ..., message = ...),
    mapperOf(".total()") { total() } to MoreThanOrEqualValidator(10)
)
```

## ValidatorDSL

As you could probably tell from above, manually creating validators, while
doable, is not super readable. TV provides a handy DSL for creating custom
validators.

Here's an example of such a validator:

```kotlin
data class Person(val name: String, val age: UInt)

val personValidator = createValidator<Person> {
    // Using reflection
    Person::name / size(isMoreThan(1))
    // Using a lambda (i.e. manually creating a mapper)
    of(".age") { age } / (isMoreThanOrEq(2) and isLessThanOrEq(130))
}
```

And here's an example with a more deeply nested object structure that contains nulls:

```kotlin
data class Person(val name: String, val age: UInt)
data class Group(val people: List<Person>)

val groupValidator = createValidator<Group?> {
    this / ifPresent {
        Group::people / {
            this / size(isIn(2..100))
            this / all {
                Person::name / size(isMoreThan(1))
                Person::age / isIn(2..130)
            }
        }
    }
}
```

### Basic structure

The idea here is that the `createValidator` function builds a `NodeValidator`. Within the `createValidator` block, you can add validators using the `/` operator.

```kotlin
createValidator<Person> {
    Person::name / isNotBlank()
//  ------------   -----------
//     Mapper       Validator
}
```

### DSL Mappers

The following are acceptable for mappers. Consider that `T` is the type of the object being validated, and `R` is the type of the mapper result.

- A [`KProperty1<T, R>`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.reflect/-k-property1/) reference (e.g. `Person::name`)
- Manually creating a mapper with `that`, e.g. `that(".name") { name }`. This avoids the use of reflection.
- Using `this`, which will add a validator based on the current object being validated.

### DSL Validators

Any regular `Validator<R>` can be used, but most built-in validators have a DSL equivalent.

### Special constructs

In addition to regular mappers and validators, the following additional constructs are available.

#### Nested validation

You can nest validators, which allows you to perform more advanced validation against an object.

You can do this by providing a lambda instead of a validator. The lambda will be used identically to the one you pass to `createValidator`, e.g.:

```kotlin
createValidator<Person> {
    Person::age / {
        this / isMoreThanOrEq(2)
        this / isLessThanOrEq(130)
    }
}

createValidator<Group> {
    Group::people / {
        this / size(isIn(2..100))
        this / all {
            Person::name / size(isMoreThan(1))
            Person::age / isIn(2..130)
        }
    }
}
```

#### Collection validation

Additional matchers exist for collections. In all cases, these would map to each element of the collection and the validator is applied to each element of the collection.

#### Validation and nullable values

Additional validators exist for dealing with nullable values:

- `ifPresent`

  - `ifPresent { ... }`: is valid if the value is null OR the value is not null and the nested validator is valid.
  - `ifPresent(...)`: is valid if the value is null OR the value is not null and the given validator is valid.

- `isRequired`
  - `isRequired { ... }`: is valid if the value is not null and the nested validator is valid.
  - `isRequired(...)`: is valid if the value is not null and the given validator is valid.
  - `isRequired()`: is valid if the value is not null.

```kotlin
data class Cat(val name: String, val collar: String?)
data class Person(val name: String, val cat: Cat?)

val personValidator = createValidator<Person> {
    Person::name / isNotBlank()
    Person::cat / ifPresent {
        Cat::name / isNotBlank()
        Cat::collar / isRequired(isNotBlank())
    }
}
```

#### Custom predicate (`validIf`)

You can also create a custom validator that uses a predicate to check if a value is valid. Here is a more advanced check that verifies that, if the given person has a partner, that that partner's partner is the original person (i.e. that the partnership is reciprocal).

```kotlin
data class Person(val name: String, val partner: Person?)

val personValidator = createValidator<Person> {
    Person::name / isNotBlank()
    Person::partner / validIf { it.partner == null || it.partner.partner == it }
}
```

You can (and should) add a custom message and constraint by providing arguments to the `validIf` call, e.g.:

```kotlin
val personValidator = createValidator<Person> {
    Person::name / isNotBlank()
    Person::partner / validIf(constraint = "Partnership must be reciprocal") {
        it.partner == null || it.partner.partner == it
    }
}
```

### DSL implementation

The DSL in itself is based on the following functions and types:

```kotlin
fun <T> createValidator<Person>(block: NodeValidationDsl<T>.() -> Unit): NodeValidator<T>()

interface ValidationDsl

interface NodeValidationDsl<T> : ValidationDsl, Buildable<NodeValidator<T>> {
    operator fun <R> div(validator: Validator<R>)
    operator fun <R> Mapper<T, R>.div(validator: Validator<R>)
    operator fun <R> KProperty1<T, R>.div(validator: Validator<R>)

    operator fun <R> Mapper<T, R>.div(block: NodeValidationDsl<R>.() -> Unit)
    operator fun <R> KProperty1<T, R>.div(validator: NodeValidationDsl<R>.() -> Unit)
}

// All DSL function builders need to be defined as extensions of the ValidationDsl
fun ValidationDsl.isNotEmpty(): NotEmptyValidator = NotEmptyValidator()
```

### Examples

#### Konform's advanced README example

```kotlin
val eventValidator = createValidator<Event> {
    Event::organizer / {
        Person::email /
            matches(".+@bigcorp.com").hint("Organizers must have a BigCorp email address")
    }

    Event::attendees / (size(lessThanOrEq(100)) and all {
        Person::name / length(moreThanOrEq(2))
        Person::age / moreThanOrEq(18).hint("Attendees must be 18 years or older")
        Person::email / ifPresent(matches(".+@.+\\..+")).hint(
            "Please provide a valid email address"
        )
    })

    Event::ticketPrices / (size(moreThanOrEq(1)) and all {
        value / ifPresent(moreThanOrEq(0.01))
    })
}
```

#### Kalidation's README example

Adapted from Kalidation's README:

```kotlin
// Initial plans for TV do not include a built-in e-mail validator
val emailValidator = createValidator<String> {
    this / matches(/* some e-mail regex */)
}

val fooValidator = createValidator<Foo> {
    Foo::bar / (isNotBlank() and isIn("GREEN", "WHITE", "RED"))
    Foo::bax / (size(moreThanOrEq(3)) and emailValidator)
    // No available or planned alternative for Foo::baz { validByScript(...) }
    this / validIf { it.validate() }
    this / validIf { it.total() >= 10 }
    // OR
    mapper(".total()") { total() } / moreThanOrEq(10)
}
```

## Comparison to other validation frameworks

| Criteria \\ Framework   | Tegral Validate |  Konform  | Kalidation |      Valiktor      |
| ----------------------- | :-------------: | :-------: | :--------: | :----------------: |
| Validation style        |       DSL       |    DSL    |    DSL     |    AssertJ-like    |
| Actively maintained?    |       N/A       |    ❌     |     ❌     |    ❌ Abandoned    |
| 3rd party dependencies¹ |     ✅ None     |  ✅ None  |    5+²     | ✅ None (for core) |
| Test frameworks support |       ❌        | ✅ Kotest |     ❌     |         ❌         |
| `java.time` support     |   ❌ Planned    |    ❌     |     ✅     |         ✅         |
| `javax.money` support   |       ❌        |    ❌     |     ❌     |         ✅         |

1. Dependencies to sibling projects (e.g. Tegral Core for Tegral Validate) and Kotlin standard libraries (e.g. Kotlin Reflect) are excluded here.
2. This includes dependencies on Jakarta EE APIs

> Author note: I actually did not know about Valiktor, and it looks quite fun. It's a shame it seemingly got abandoned :c

## Future plans

TODO

### Additional built-in validators

TODO

- java.time validators
- Function return validators

## Appendix

### Table of planned validators

|                   Class                    | DSL                                                       | Parameters                 | Boolean equivalent                                            | Type restrictions |
| :----------------------------------------: | --------------------------------------------------------- | -------------------------- | ------------------------------------------------------------- | ----------------- |
|           **SIMPLE VALIDATORS**            | ---------                                                 | -----                      | ----                                                          | ----              |
|              `EqualValidator`              | `equals(ref)` (also `eq`)                                 | Reference value            | `candidate == reference`                                      |
|            `NotEqualValidator`             | `notEquals(ref)` (also `neq`)                             | Reference value            | `candidate != reference`                                      |
|              `SameValidator`               | `sameAs(ref)`                                             | Reference value            | `candidate === reference`                                     |
|             `NotSameValidator`             | `notSameAs(ref)`                                          | Reference value            | `candidate !== reference`                                     |
|              `NullValidator`               | `isNull()`                                                |                            | `candidate == null`                                           |
|             `NotNullValidator`             | `isNotNull()`, `isRequired()`                             |                            | `candidate != null`                                           |
|               `IsValidator`                | `isOf<T>()`                                               | Reference type `T`         | `candidate is T`                                              |
|              `IsNotValidator`              | `isNot<T>()`                                              | Reference type `T`         | `candidate !is T`                                             |
|            `LessThanValidator`             | `lessThan(ref)` (also `lt`, `emax`)                       | Reference value            | `candidate < reference`                                       | `Comparable`      |
|         `LessThanOrEqualValidator`         | `lessThanOrEquals(ref)` (also `leq`, `max`)               | Reference value            | `candidate <= reference`                                      | `Comparable`      |
|           `GreaterThanValidator`           | `greaterThan(ref)` (also `gt`, `emin`)                    | Reference value            | `candidate > reference`                                       | `Comparable`      |
|       `GreaterThanOrEqualValidator`        | `greaterThanOrEquals(ref, precision)` (also `geq`, `min`) | Reference value            | `candidate >= reference`                                      | `Comparable`      |
|       `EqualWithPrecisionValidator`        | `equals(ref, precision)` (also `eq`)                      | Reference value, precision | `candidate in (reference +- precision)`                       | `Number`          |
|      `NotEqualWithPrecisionValidator`      | `notEquals(ref, precision)` (also `neq`)                  | Reference value, precision | `candidate !in (reference +- precision)`                      | `Number`          |
|      `LessThanWithPrecisionValidator`      | `lessThan(ref, precision)` (also `lt`, `emax`)            | Reference value, precision | `candidate < (reference +- precision)`                        | `Number`          |
|  `LessThanOrEqualWithPrecisionValidator`   | `lessThanOrEquals(ref, precision)` (also `leq`, `max`)    | Reference value, precision | `candidate <= (reference +- precision)`                       | `Number`          |
|    `GreaterThanWithPrecisionValidator`     | `greaterThan(ref, precision)` (also `gt`, `emin`)         | Reference value, precision | `candidate > (reference +- precision)`                        | `Number`          |
| `GreaterThanOrEqualWithPrecisionValidator` | `greaterThanOrEquals(ref, precision)` (also `geq`, `min`) | Reference value, precision | `candidate >= (reference +- precision)`                       | `Number`          |
|         **COMBINATION VALIDATORS**         | ---------                                                 | -----                      | ----                                                          | ----              |
|            `AllValidValidator`             | `and(...)`, `(... and ...)`                               | 0+ validators              | (AND) `validators.all { it.isValid(candidate) }`              |                   |
|            `AnyValidValidator`             | `or(...)`, `(... or ...)`                                 | 0+ validators              | (OR) `validators.any { it.isValid(candidate) }`               |                   |
|            `NotValidValidator`             | `not(...)`                                                | 1 validator                | (NOT) `!validator.isValid(candidate)`                         |                   |
|          `OnlyOneValidValidator`           | `xor(...)`, `(... xor ...)`                               | 0+ validators              | (XOR) `validators.count { it.isValid(candidate) } == 1`       |                   |
|         `NotNullAndValidValidator`         | `isRequired(...)`, `isRequired { ... }`                   | 1 validator                | `candidate != null && validator.isValid(candidate)`           |                   |
|           `NullOrValidValidator`           | `ifPresent(...)`, `ifPresent { ... }`                     | 1 validator                | `candidate == null`                                           |                   |
|        `CustomConstraintValidator`         | `.hint("...")`                                            | 1 validator                | `validator.isValid(candidate)` with custom constraint message |                   |
|         **COLLECTION VALIDATORS**          | ---------                                                 | -----                      | ----                                                          | ----              |
|              `SizeValidator`               | `size(...)`                                               | 1 validator                | `validator.isValid(candidate.size)`                           | Collections       |
|           `AllElementsValidator`           | `all { ... }`, `all(...)`                                 | 1 validator                | `candidate.all { validator.isValid(it) }`                     | Collections       |
|           `AnyElementValidator`            | `any { ... }`, `any(...)`                                 | 1 validator                | `candidate.any { validator.isValid(it) }`                     | Collections       |
|         `OnlyOneElementValidator`          | `onlyOne { ... }`, `onlyOne(...)`                         | 1 validator                | `candidate.count { validator.isValid(it) } == 1`              | Collections       |
|    **CHARSEQUENCE (STRING) VALIDATORS**    | ---------                                                 | -----                      | ----                                                          | ----              |
|             `LengthValidator`              | `length(...)`                                             | 1 validator                | `validator.isValid(candidate.length)`                         | `CharSequence`    |
|         `MatchesPatternValidator`          | `matches(...)`                                            | 1 regex                    | `candidate.matches(regex)`                                    | `CharSequence`    |
|            `NotBlankValidator`             | `notBlank()`                                              |                            | `candidate.isNotBlank()`                                      | `CharSequence`    |
|           `NullOrBlankValidator`           | `nullOrBlank()`                                           |                            | `candidate == null`                                           |                   |
|              `BlankValidator`              | `blank()`                                                 |                            | `candidate.isBlank()`                                         | `CharSequence`    |
|            `NotEmptyValidator`             | `notEmpty()`                                              |                            | `candidate.isNotEmpty()`                                      | `CharSequence`    |
|           `NullOrEmptyValidator`           | `nullOrEmpty()`                                           |                            | `candidate == null`                                           |                   |
|              `EmptyValidator`              | `empty()`                                                 |                            | `candidate.isEmpty()`                                         | `CharSequence`    |
|            **MISC. VALIDATORS**            | ---------                                                 | -----                      | ----                                                          | ----              |
|              `NodeValidator`               | `{ ... }`                                                 | Mapped validators          | `mv.all { (m, v) -> v.isValid(m(candidate)) }`                |                   |
|            `PredicateValidator`            | `validIf { ... }`                                         | `(T) -> Boolean`           | `predicate(candidate)`                                        |                   |
|             `InRangeValidator`             | `isIn(...)`                                               | Range                      | `candidate in range`                                          |                   |
|           `NotInRangeValidator`            | `isNotIn(...)`                                            | Range                      | `candidate !in range`                                         |                   |
|          `InCollectionValidator`           | `isIn(...)`                                               | Collection                 | `candidate in collection`                                     |                   |
|         `NotInCollectionValidator`         | `isNotIn(...)`                                            | Collection                 | `candidate !in collection`                                    |                   |

## TODO in this spec

- More than -> greaterThan
- I18n support for messages
- Combinations in ValidatorDSL
- `key`/`value` map entry mapper, similar thing for pairs, etc.
- check that every example matches the appendix, there might have been some drift during design
