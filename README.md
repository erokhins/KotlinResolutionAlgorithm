# KotlinResolutionAlgorithm
Description for new resolve algorithm

## ExpressionContext
This context contains:
- Scope
- Expected type
- Resolve mode(ONLY_RESOLUTION or RESOLUTION_WITH_INFERENCE)

## Expression Typing Facade
For each expression with ExpressionContext we can get JetTypeInfo.
If Resolve mode is RESOLUTION_WITH_INFERENCE then JetTypeInfo contains the following information:
- JetType for the expression
- data flow info

Otherwise, if resolve mode is ONLY_RESOLUTION, JetTypeInfo contains:
- data flow info
- additional data flow info.
- constraint system for this expression
	- call tree for inner calls
	- some info about inner lambdas and callable references

Examples.
```Kotlin
fun test(): Int

val a = test()
```
For `test()` expression we created context with NO_EXPECTED_TYPE and RESOLUTION_WITH_INFERENCE, because it is a top-level call.

Result was: JetType = Int, empty data flow info.

```Kotlin
fun emptyList<E>(): List<E>

val list: List<Int> = emptyList()
```

For `emptyList()` context has expected type `List<Int>` and the same resolve mode: RESOLUTION_WITH_INFERENCE.
But we always do resolve with ONLY_RESOLUTION, after that, if real resolve mode is RESOLUTION_WITH_INFERENCE
we complete inference i.e. solve constraint system.

Result for the first stage:  
- Constraint system: 
	- Type variable: `E`
	- return type `List<E>`
	- E <: Int


> TODO: Remove addition data flow info after resolve function body

## Call Resolver

*Input:*
- call `[r].foo(a_1, a_2, ..., l_1, l_2, ... cr_1, cr_2 , ...)`
	- receiver may not be present
	- l_1, l_2... are lambda arguments;
	- cr_1, cr_2... - callable references
	- order of arguments can be any
- CallContext - ExpressionContext with some additional info

*Output:*
- resolution result
- JetTypeInfo

The process of the call resolution contains steps that are described below:

1. Resolve receiver and arguments. Collect their type infos.
2. Build prioritized tasks. Each task contains several candidates, all of them having the same priority.
3. Resolve(try) each candidate. Remove unsuccessful candidates.
4. Find the most specific candidate.
5. If resolve mode is RESOLUTION_WITH_INFERENCE then solve constaint system.


## Resolve receiver and arguments
We analyze receiver and all regular arguments a_1, ... and save their JetTypeInfo.

We also analyze all callable references cr_1, cr_2..., but here are two cases:

1. There exists only one callable reference with given receiver and function name.
	- In this case we have to deal with such callable reference as with a regular argument
	
2. There are several callable references:
	- Then we have to deal with it as with a lambda.
	
```Kotlin
fun bas(s: String): Int {}

fun foo() {}
fun foo(a: Int) {}

val a = bar(::bas, ::foo)
``` 
For argument `::bas` we have only one candidate `bas(String): Int`. Therefore, `::bas` has JetTypeInfo with type `(String) -> Int`.

*NOTE:* If class A has member function `foo` and extension function `foo` then `x::foo` has at least two candidates.

But argument `::foo` has two candidates. Thereby, we can't get JetTypeInfo for this argument.
We have to do with this argument like `{ arguments -> foo(arguments) }`, where arguments with their types get out after name resolution for `bar` function.

*NOTE:* If receiver is callable reverence then we always have to deal with it as with a regular argument.

For function literal arguments we constrain shape types.
Example: for lambda `{ x: Int, y ->  some code }` a shape type is \[(Int, ???) -> ??? or ???.(Int, ???) -> ???\].

*NOTE:* We can't express receiver type for lambda expression. Also we can't figure out precise function type for lambda.

> TODO: `val a = { 5 }` what type has `a`?


#### Data flow info for arguments

Data flow info for receiver must be considered for arguments. Also data flow info for the first argument must be considered for the second argument, etc.

> TODO: for call `foo(arg2 = ..., arg1= ...)` data flow info counted in the wrong order.

Examples:
```Kotlin
fun foo(a: () -> Int, b: Int, c: () -> Int) {}

fun test(x: Int?){
	x?.plus(x) // x in argument is not null

	foo({x}, x!!, {x}) // both smart casts are correct	
}
```

As we see above, data flow info for lambda contains all data flow info for regular arguments. 
It is correct, because at runtime we calculate all arguments before running lambda(for them we just create anonymous object)

```Kotlin
class X {
	fun foo() {}
	fun foo(i: Int) {}
}
fun foo(() -> Unit, x: X) {}

fun test(x: X?) {
	foo(x!!::foo, x) // smart cast for second argument
}
```
Data flow info for receiver of callable reference must be considered for following arguments even if callable reference has several candidates.

Data flow info must be taken into account when we construct constraint system:
```Kotlin
fun <T> proxy(t: T): T = t

fun bar(a: Int) {}

fun test(x: Int?) {
    if (x != null) {
        val a = proxy(x) // smart cast, a: Int
        
        bar(proxy(x)) // smart cast
    }
}
```

## Build prioritized tasks

If call `foo` has receiver `r`, then `r` has JetTypeInfo with type for `r`. 
Its type may contain nonfixed type variables T_i. Also JetTypeInfo contains constraint system for T_i.

When we create task(group of candidates) with members of `r`, we should collect the following candidates:

- If `r` has type like `List<T>`, then we just collect all members of `List<X>` with substituted type parameters: `X = T`.
- If `r` has type `T`, where `T` is type variable, then we get all constraints like `T` is a subtype of `SomeType`, and collect all members from all `SomeType`.

Example. Let `T` have the following constraints: `T <: X, T <: BarU, T = BarE, T :> BarL, T <: Comparable<T>`. (X is a type variable)
Then we collect all members with name `foo` from `BarU`, `BarE`, `Comparable<E>`.

When we created task with extension function, we just collected all extension function with given name.

> TODO: some optimization for extension function?

> TODO: add full description for task creation: Impicit receiver, member|extension receivers, invoke convention.

## Lambdas
We can't analyze lambda body until we know all types of arguments and receiver.
Therefore we begin the process of body analyze when we know all types. 
This may happen for one of the following reasons:

- Function, which receive lambda, has explicit types for lambda parameters and receiver.
- We solved part(or all) of constraint system and parameter types have become known.

For the last expression in lambda body and for all labeled returns resolve mode was ONLY_RESOLUTION.
If return type for lambda is known, then we complete analyze with given expected type. 
Otherwise, we combine constraint systems for these expression and for upper calls. 

Examples:
```Kotlin
fun <K, V> bar(k: K, run: (K) -> V): V = null!! 
fun listOf<T>(value: T): List<T> = null!!

val a = bar(1) { // after we analyzed expression `1`, we fixed K = Int
    if (it == 0) return@bar listOf(it) // JetTypeInfo: List<T1>, T1 :> Int

    listOf("") // JetTypeInfo: List<T2>, T2 :> String
} 
/*
Common system:
V :> List<T1>
V :> List<T2>
T1 :> Int
T2 :> String
*/
```

## Resolve candidate
For each candidate we must decide if it is a match or not. If not, we collect some errors.

Note that we already collected JetTypeInfo for arguments and receiver. 
Also we built shape types for lambda arguments(see *Resolve receiver and arguments*).

Now we construct common system which contains type variable for call `foo` and all constraint system for receiver and arguments.
After that, we search for conflicts in this system(i.e. system never succeeded). 
If we haven't found them, then candidate is successful and JetTypeInfo contains this common system.

*WARNING:* We must never run lambda analyze in this process, because we must analyze lambda only onсe.

> TODO: Arguments to parameters mapping, shape types checks.

## Order for solving constraint system
We create special order for fixing type variables.
```Kotlin
fun <T> ref(s: T): Int = 1
fun ref(s : String) {}
fun <T, K> foo(f: (T) -> K, g: (K) -> Int): T = null!!

fun test1() {
	val a: Int = foo ( ::ref ) { it }
}
/*
Constraint system for foo:
T <: Int
K (empty)
Fix T = Int & resolve ::ref to <T> ref(T): Int 

new constraint system for foo:
K >: Int
Fix K = Int & resolve { it }
*/
```

----

> TODO: 
>	
> - Most specific candidate.
> - Constrain system.
> - Conventions: plus, invoke.
> - Special calls: `if`, `when`, `!!`, `?:`. 

## Constraint system optimization. Similar systems
```Kotlin
val a = listOf(
    Pair('a', mapOf()),
    Pair('b', mapOf()),
    Pair('c', mapOf()),
    Pair('d', mapOf(Pair("", ""))),
    Pair('e', mapOf())
)
``` 
For this case we can detect that `Pair('a', mapOf())` and `Pair('b', mapOf())` are similar.
Or more precisely, constraint systems for both these calls are equal, and we can add only one of them to constraint system for `listOf`.

About equals (and hashcode) for constraint system.
For each constraint system we can rename type variables in a special way. 
Example: `foo<T1, T2>(bar<T3, T4>(bas<T5>()), bal<T6>())`.
After that we can collect original constraints with new names for type variables, and concat all constraints to string.
If these two strings are equal then constraint systems are equal.
