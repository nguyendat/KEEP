# Guards

* **Type**: Design proposal
* **Author**: Alejandro Serrano
* **Contributors**: Mikhail Zarechenskii
* **Discussion**: ??

## Abstract

We propose an extension of branches in `when` expressions with subject, which unlock the ability to perform multiple checks in the condition of the branch.

## Table of contents

* [Abstract](#abstract)
* [Table of contents](#table-of-contents)
* [Motivating example](#motivating-example)
  * [Clarity over power](#clarity-over-power)
  * [One little step](#one-little-step)
* [Technical details](#technical-details)
  * [Alternative syntax](#alternative-syntax)
* [Exhaustiveness checking](#exhaustiveness-checking)
* [Potential extensions](#potential-extensions)

## Motivating example

Consider the following types, which model the state of a UI application.

```kotlin
enum class Problem {
    CONNECTION, AUTHENTICATION, UNKNOWN
}

sealed interface Status {
    data object Loading: Status
    data class Error(val problem: Problem, val isCritical: Boolean): Status
    data class Ok(val info: List<String>): Status
}
```

Kotlin developers routinely use [`when` expressions](https://kotlinlang.org/docs/control-flow.html#when-expression)
with subject to describe the control flow of the program.

```kotlin
fun render(status: Status): String = when (status) {
    Status.Loading -> "loading"
    is Status.Error -> "error: ${status.problem}"
    is Status.Ok -> status.info.jointToString()
}
```

Note how [smart casts](https://kotlinlang.org/docs/typecasts.html#smart-casts) interact with
`when` expressions. In each branch we get access to those fields available only on the
corresponding subclass of `Status`.

Using a `when` expression with subject has one important limitation, though: each branch must
depend on a _single condition over the subject_. _Guards_ remove this limitation, so you
can introduce additional conditions, separated with `&&` from the subject condition.

```kotlin
fun render(status: Status): String = when (status) {
    Status.Loading -> "loading"
    is Status.Ok && status.info.isEmpty() -> "no data"
    is Status.Ok -> status.info.joinToString()
    is Status.Error && status.problem == Problem.CONNECTION ->
      "problems with connection"
    is Status.Error && status.problem == Problem.AUTHENTICATION ->
      "could not be authenticated"
    else -> "unknown problem"
}
```

The combination of guards and smart casts gives us the ability to look "more than one layer deep".
For example, after we know that `status` is `Status.Error`, we get access to the `problem` field.

The code above can be rewritten using nested control flow, but "flattening the code" is not the
only advantage of guards. They can also simplify complex control logic. For example, the code
below `render`s both non-critical problems and success with an empty list in the same way.

```kotlin
fun render(status: Status): String = when (status) {
    Status.Loading -> "loading"
    is Status.Ok && status.info.isNotEmpty() -> status.info.joinToString()
    is Status.Error && status.isCritical -> "critical problem"
    else -> "problem, try again"
}
```

Such code is harder to describe using a subject and nested control flow. It is possible to switch
to a `when` expression without subject, but that obscures the fact that our control flow depends
on the `Status` value.

```kotlin
fun render(status: Status): String = when {
    status == Status.Loading -> "loading"
    status is Status.Ok && status.info.isNotEmpty() -> status.info.joinToString()
    status is Status.Error && status.isCritical -> "critical problem"
    else -> "problem, try again"
}
```

The Kotlin compiler provides more examples of this pattern, like
[this one](https://github.com/JetBrains/kotlin/blob/112473f310b9491e79592d4ba3586e6f5da06d6a/compiler/fir/resolve/src/org/jetbrains/kotlin/fir/resolve/calls/Candidate.kt#L188)
during candidate resolution.

### Clarity over power

One important design goal of this proposal is to make code _clear_ to write and read.
For that reason, we have consciously removed some of the potential corner cases.
Furthermore, we are conducting a survey to ensure that guards are intuitively understood
by a wide range of Kotlin developers.

First, it is not allowed to mix several conditions (using `,`) and guards in a single branch.
They can be used in the same `when` expression, though.
Here is one example taken from [JEP-441](https://openjdk.org/jeps/441).

```kotlin
fun testString(response: String?) {
    when (response) {
        null -> { }
        "y", "Y" -> println("You got it")
        "n", "N" -> println("Shame")
        else if response.equals("YES", ignoreCase = true) ->
          println("You got it")
        else if response.equals("NO", ignoreCase = true) ->
          println("Shame")
        else -> println("Sorry")
    }
}
```

This example also shows that the combination of `else` with a guard
-- in other words, a branch that does not inspect the subject --
is done with special `else if` syntax.

Second, we do not allow disjunctions (`||`) directly in a guard.
The following is an example of a branch condition that may be
difficult to understand, because `||` usually has lower priority
than `&&` when used in expressions.

```kotlin
is Status.Success && info.isEmpty || status is Status.Error
// would be equivalent to
is Status.Success && (info.isEmpty || status is Status.Error)
```

This does not mean that you cannot use disjunctions at all in guards.
You are just required to introduce extra parentheses.

### One little step

Whereas other languages have introduced full pattern matching
(like Java in [JEP-440](https://openjdk.org/jeps/440) and [JEP-441](https://openjdk.org/jeps/441)),
we find guards to be a much smaller extension to the language,
but covering many of those use cases. This is possible because
we benefit from the powerful data- and control-flow analysis in
the language. If we perform a type test on the subject, then
we can access all the fields corresponding to that type in the guard.

## Technical details

We extend the syntax of `whenEntry` in the following way.

```
whenEntry : whenCondition [{NL} whenEntryAddition] {NL} '->' {NL} controlStructureBody [semi]
          | 'else' [ 'if' expression ] {NL} '->' {NL} controlStructureBody [semi]

whenEntryAddition: ',' [ {NL} whenCondition { {NL} ',' {NL} whenCondition} ] 
                 | '&&' {NL} conjunction
```

Entries with guards (that is, using `&&` or `else if`) may only appear in `when` expressions with a subject.

We use `equality` instead of `expression` in the second line to prevent ambiguities; as a result, parentheses are required when a disjunction is used as guard expression.

```kotlin
when (s) {
    is String && (s == "" || s == "no") -> ...
}
```

The behavior of a `when` expression with guards is equivalent to the same expression in which the subject has been inlined in every location,
and `else if` has been replaced with the guard expression itself. The first version of the motivating example is equivalent to:

```kotlin
fun render(status: Status): String = when {
    status == Status.Loading -> "loading"
    status is Status.Ok && status.info.isEmpty() -> "no data"
    status is Status.Ok -> status.info.joinToString()
    status is Status.Error && status.problem == Problem.CONNECTION ->
      "problems with connection"
    status is Status.Error && status.problem == Problem.AUTHENTICATION ->
      "could not be authenticated"
    else -> "unknown problem"
}
```

The current rules for smart casting imply that any data- and control-flow information gathered in the left-hand side of `&&` is available on the right-hand side (for example, when you do `list != null && list.isNotEmpty()`).

### Alternative syntax

We have considered other potential syntax for guards.
We are conducting a survey to gather the opinions of a wide range of Kotlin developers,
in order to inform our final decision.

- Uniformly using `&&`: `else && condition` looks quite alien, `else if` is a well-known concept.
- Uniformly using `if`:
  - Pro: delineates the guard much more clearly than `&&`.
  - Con: writing several conditions in `if` or a `when` without subject is already done with `&&`.
- Using another keyword:
  - Similar to uniformly using `if`, with the potential disadvantage of having a new keyword.

## Exhaustiveness checking

Cases like `Status::render` above point out that we may want more powerful exhaustiveness checking that we have now, which is only limited to the subject of `when`.
For example, if we forget the last `else` in the example introduced at the beginning of this proposal, we should get:

```kotlin
error: 'when' expression must be exhaustive.
Add the necessary 'is Error && status.problem == UNKNOWN' branch,
or 'else' branch instead.
```

Although not strictly part of this proposal, we have created a [sibling issue](https://youtrack.jetbrains.com/issue/KT-63696/Improved-exhaustiveness-checking)
to improve exhaustiveness checking in several directions, including the aforementioned one.
The core of that improved algorithm is to consider not only the subject of the `when` expression, but also any stable reference mentioned in the branches.

## Potential extensions

We have considered an extension to provide access to members in the subject within the condition.
For example, you would not need to write `status.info.isNotEmpty()` in the second of the examples.

```kotlin
fun render(status: Status): String = when (status) {
    Status.Loading -> "loading"
    is Status.Ok && info.isNotEmpty() -> status.info.joinToString()
    is Status.Error && isCritical -> "critical problem"
    else -> "problem, try again"
}
```

We have decided to drop this extension for the time being for three reasons:

- It is somehow difficult to explain that you can drop `status.` from the condition, but not in the body of the branch (like the second branch above shows). However, being able to drop `status.` also in the body would be a major, potentially breaking, change for the language.
- It is not clear how to proceed when one member of the subject has the same name as a member in scope (from a local variable or from `this`).
- The simple translation from guards to `when` without subject explained in the _Technical details_ section is no longer possible.
