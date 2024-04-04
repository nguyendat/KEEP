# Better escaping of `$` and `"`

* **Type**: Design proposal
* **Authors**: Alejandro Serrano Mena
* **Discussion**: ??
* **Prototype**: Implemented in [this branch](https://github.com/JetBrains/kotlin/compare/rr/serras/escape-dollar)

## Abstract

We propose an extension of string literal syntax to improve the situation around escaping `$` and `"`, especially in multiline strings.

## Table of Contents

* [Abstract](#abstract)
* [Table of Contents](#table-of-contents)
* [Motivating examples](#motivating-examples)
* [Proposed solution](#proposed-solution)
    * [Single-line string literals](#single-line-string-literals)

## Motivating examples

Strings are one of the fundamental types in Kotlin, developers routinely create (parts of) them by using string literals. However, the current design has a few inconveniences, as witnessed by this [YouTrack issue](https://youtrack.jetbrains.com/issue/KT-2446/String-literals). This KEEP pertains to how to improve the situation around escaping `$` and `"`, especially in multiline strings. It is a non-goal to change the behavior regarding indentation (or stripping thereof).

[Kotlin's multiline strings](https://kotlinlang.org/docs/strings.html#multiline-strings) are raw, that is, every character from the start to the end markers is taken as it appears. In particular, there are no escaping sequences (`\n`, `\t`, ...) as found in single line strings. Still, `$` is used to mark interpolation, and `""""` is used to mark the end of the string. If you need those characters in the string, the most often used workaround is to interpolate the character, leading to an awkward sequence of characters.

```kotlin
val s = """
        ${order.price}${'$'}
        """
```

This workaround has additional (bad) consequences if in the future Kotlin implements a feature akin to string templates. That `'$'` character would appear as one of the interpolated values, instead of as "static part" of the string.

## Proposed solution

> The solution is inspired by [C# 11's raw string literals](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-11.0/raw-string-literal#detailed-design-interpolation-case). 

Every multiline string literal **begins** with a sequence of zero or more `$` symbols, followed by a sequence of at least three `"` symbols.

**Interpolation** is done using the same amount of `$` symbols as the one heading the string.

* Or with exactly one `$` if none are heading, which is the current behavior.

Marking the **end of the string** is done using as many `"` symbols as those beginning the string.

* This allows using more than three consecutive quote symbols within the string.

Note that multiline strings already allow one or two quote symbols within the string.

```kotlin
val p = """$actor said "$message"."""
```

### Single-line string literals

Single-line strings have their own ways of escaping those symbols, using a backslash,

```kotlin
val r = "$thing costs $price\$"
```

For the sake of uniformity, we extend single-line string literals with the `$` escaping policy described above.

```kotlin
val r = $$"$$thing costs $$price$"
```

This feature some use cases, like [better interoperability with i18n software](https://youtrack.jetbrains.com/issue/KT-7258/String-interpolation-plays-badly-with-i18n-and-string-positioning).