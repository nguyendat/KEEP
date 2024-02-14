# Guards

* **Type**: Design proposal
* **Author**: Alejandro Serrano
* **Contributors**: Mikhail Zarechenskii
* **Discussion**: [KEEP-371](https://github.com/Kotlin/KEEP/issues/371)

## Abstract

We propose an extension of branches in `when` expressions with subject, which unlock the ability to perform multiple checks in the condition of the branch.

## Table of contents

* [Abstract](#abstract)
* [Table of contents](#table-of-contents)
* [Motivating example](#motivating-example)
  * [Clarity over power](#clarity-over-power)
  * [One little step](#one-little-step)
* [Technical details](#technical-details)
  * [The need for `else`](#the-need-for-else)
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

> In the following examples we use `&&`, but the syntax is not yet definitive.
> Another option is using `if` as the keyword introducing the guard.

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

If we rewrite the code above using a subject and nested control flow, we need to
_repeat_ some code. This hurts maintainability since we need to remember to 
keep the two `"problem, try again"` in sync.

```kotlin
fun render(status: Status): String = when (status) {
    Status.Loading -> "loading"
    is Status.Ok ->
      if (status.info.isNotEmpty()) status.info.joinToString()
      else "problem, try again"
    is Status.Error ->
      if (status.isCritical) "critical problem"
      else "problem, try again"
}
```

It is possible to switch to a `when` expression without subject,
but that obscures the fact that our control flow depends
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

Second, if we choose `&&` as the keyword introducing a guard,
we do not allow disjunctions (`||`) in the expression afterward.
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
                 | '&&' {NL} conjunction  // using '&&'
                 | 'if' {NL} expression   // using 'if'
```

Entries with guards (that is, using `&&`/`if` or `else if`) may only appear in `when` expressions with a subject.

We use `conjunction` instead of `expression` in the case of `&&` to prevent ambiguities; as a result, parentheses are required when a disjunction is used as guard expression.

```kotlin
when (s) {
    is String && (s == "" || s == "no") -> ...
}
```

The behavior of a `when` expression with guards is equivalent to the same expression in which the subject has been inlined in every location, `if` has been replaced by `&&`, and `else` by `true` (or more succintly, `else if` is replaced by the expression following it). The first version of the motivating example is equivalent to:

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

The current rules for smart casting imply that any data- and control-flow information gathered in the left-hand side is available on the right-hand side (for example, when you do `list != null && list.isNotEmpty()`).

### The need for `else`

The current document proposes using `else if` when there is no matching in the subject, only a side condition.
One promising avenue is to drop the `else`, or even the whole `else if` entirely.
Alas, this creates a problem with the current grammar, which allows _any_ expression to appear as a condition.

```kotlin
fun weird(x: Int) = when (x) {
  0 -> "a"
  if (something()) 1 else 2 -> "b"
  else -> "c"
}
```

### Alternative syntax

We have considered other potential syntax for guards.
We are conducting a survey to gather the opinions of a wide range of Kotlin developers,
in order to inform our final decision.

- Uniformly using `&&`: `else && condition` looks quite alien, `else if` is a well-known concept.
- Uniformly using `if`:
  - Pro: delineates the guard much more clearly than `&&`.
  - Con: writing several conditions in `if` or a `when` without subject is currently done with `&&`.
  
      ```kotlin
      when (x) { is List if x.isEmpty() -> ... }
      // as opposed to
      if (x is List && x.isEmpty()) { ... }
      when { x is List && x.isEmpty() -> ... }
      ```
- Using `when`:
  - Pro: they are already keywords, so the change to the grammar is easy.
  - Pro: similar to Java and C#.
- Using another keyword:
  - Similar to uniformly using `if`, with the potential disadvantage of having a new keyword.

## Exhaustiveness checking

Cases like `Status::render` above point out that we may want more powerful exhaustiveness checking that we have now,
which is only limited to the subject of `when`.
For example, if we forget the last `else` in the example introduced at the beginning of this proposal, we should get:

```kotlin
error: 'when' expression must be exhaustive.
Add the necessary 'is Error && status.problem == UNKNOWN' branch,
or 'else' branch instead.
```

We propose extending the current exhaustiveness check analysis to account not only for the type of the subject
but also every [stable reference](https://kotlinlang.org/spec/type-inference.html#smart-cast-sink-stability)
appearing on it. That includes immutable properties, among others.

In the following description, we assume that every `when` expression has been written or
translated to a form in which each branch is headed by a conjunction of expressions, or
is simply `else`.

```kotlin
when {
  e1 && e2 && ... ->
  f1 && f2 && ... ->
  ...
}
```

The exhaustiveness check may consider only those branches in which _every_ condition is of
one of the following forms.

- `r == <literal>` or `r == <enum entry>`,
- `r is Type`,
- `r == null` or `r != null`,
- `r` or `!r`, if `r` is of Boolean type,

and where `r` is a _reference_, that is, is either the original subject of the `when` expression,
or a stable reference.

If should then proceed to check that the cartesian product of all the potential types of the
references are _covered_, as per the current exhaustiveness check. For example, the following should
be considered exhaustive:

```kotlin
fun f(x: Int?, y: Int?): Int = when {
  x == null && y == null -> 0
  x != null && y == null -> x
  x == null && y != null -> y
  x != null && y != null -> x + y
}
```

This proposal does not mandate a particular implementation. However, this is a potential algorithm.
First, we extract all the references in the remaining expressions and perform a topological sort,
so that `x` appears before `x.property`. Then we execute `coveredBy(references, branchConditions)`,
given by the following pseudocode.

```kotlin
coveredBy(<empty list>, _) = OK
coveredBy(r + rest, branchConditions) {
  val useful = branchConditions.filter { r appears }
  val groups = group(branchConditions) { condition on r }

  val coveredConditions = mutableListOf()
  for ((condition, group) in groups) {
    val smallerConditions = group.map { remove condition from it }
    if coveredBy(rest, smallerConditions) {
      coveredConditions.add(condition)
    }
  }

  if !(coveredCondition covers type(r)) FAIL
}
```

The previous example using `x` and `y` would proceed as follows:

```
coveredBy([x, y], [x == null && y == null, ...])
-> go with r = x
  -> group with x == null
  |  smallerConditions = [ y == null, y != null ]
  |
  | -> coveredBy([y], [ y == null, y != null ])
  |   -> go with r = y
  |   | -> group with y == null
  |   |    smallerConditions = [] -> OK
  |   | -> group with y != null
  |   |    smallerConditions = [] -> OK
  | 
  | -> add x == null to coveredConditions

  -> group with x != null
  |  smallerConditions = [ y == null, y != null ]
  |
  | -> coveredBy([y], [ y == null, y != null ])
  |    // same as above
  | -> add x != null to coveredConditions

  -> does [ x == null, x != null ] cover `Int?`
  | -> YES
  -> return OK
```

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
