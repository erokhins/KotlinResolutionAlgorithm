# KotlinResolveAlgorithm
Description for new resolve algorithm

### Expression Typing Facade

Computes a type for an expression.

- Input:
  - expression
  - context (ExpressionTypingContext)
- Output:
  - TypeInfo for the expression

> Code: ExpressionTypingFacade.getTypeInfo()

### Type Info

For each expression we collect type info, that means “what type does this expression have independently of context”. By context dependency we call the dependency of the expected type for this expression.
After the expected type for the expression becomes known, we complete analysis of  the expression according to it.

Example. 
``` Kotlin
fun emptyList<E>(): List<E>

val list: List<Int> = emptyList()
```

Here the type of the expression `emptyList()` is `List<Int>`, but we know it because we declared it explicitly as a type of a variable (and therefore it became an expected type for the expression). If we examine this expression independently of context, it'll have the type info `List<E>` where `E` is unknown.

Another example.

``` Kotlin
fun listOf<E>(vararg elements: E): List<E>

val list: List<Int> = listOf(1, “a”)
```

Type info of the expression `listOf(1, “a”)` is `List<E>` where there are constraints on type `E` derived from actual arguments `(1, “a”)`:
E is a supertype of Int
E is a supertype of String
But if we consider this expression in its context with expected type List<Int>, it occurred to be not typeable, because expected type adds a constraint 
E is a subtype of Int
that makes the constraints incompatible.

TypeInfo for an expression is:
type that might have unknown yet type arguments ();
data flow info;
constraint system that imposes constraints on type arguments  as well as some other variables; system might be empty.

> Code: JetTypeInfo
