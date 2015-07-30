# KotlinResolveAlgorithm
Description for new resolve algorithm

## ExpressionContext
This context contains:
- Scope
- Expected type
- Resolve mode(ONLY_RESOLUTION or RESOLUTION_WITH_INFERENCE)

## Expression Typing Facade
For each expression with ExpressionContext we can get JetTypeInfo.
If Resolve mode is RESOLUTION_WITH_INFERENCE then JetTypeInfo contains that information:
- JetType for expression
- data flow info

Otherwise, if resolve mode is ONLY_RESOLUTION, JetTypeInfo contains:
- data flow info
- addition data flow info.
- constrain system for this expression
	- call tree for inner calls
	- some info about inner lambdas and callable references

Examples.
```Kotlin
fun test(): Int

val a = test()
```
For `test()` expression we created context with NO_EXPECTED_TYPE and RESOLUTION_WITH_INFERENCE, because it is top-level call.

Result was: JetType = Int, empty data flow info.

```Kotlin
fun emptyList<E>(): List<E>

val list: List<Int> = emptyList()
```

For `emptyList()` context have expected type `List<Int>` and same resolve mode: RESOLUTION_WITH_INFERENCE.
But we always do resolve with ONLY_RESOLUTION, after that, if real resolve mode is RESOLUTION_WITH_INFERENCE
we complete inference i.e. solve constrain system.

Result for first stage:  
- Constrain system: 
	- Type variable: `E`
	- return type `List<E>`
	- E <: Int


> TODO: Remove addition data flow info after resolve function body

## Call Resolver

*Input:*
- call `[r].foo(a_1, a_2, ..., l_1, l_2, ... cr_1, cr_2 , ...)`
	- receiver may not exist
	- l_1, l_2... are lambda arguments;
	- cr_1, cr_2... - callable references
	- order of arguments can be any
- CallContext - ExpressionContext with some addition info

*Output:*
- resolution result
- JetTypeInfo

The process of the call resolution contains steps that are descripted below:

1. Resolve receiver and arguments. Collect their type infos.
2. Build prioritize task. Each task contains several candidates, which have same priority.
3. Resolve(try) each candidate. Remove unsuccessful candidates.
4. Find the most specific candidate.
5. If resolve mode is RESOLUTION_WITH_INFERENCE then solve constain system.


## Resolve receiver and arguments
We analyze receiver and all usual arguments a_1, ... and save their JetTypeInfo.

We also analyze all callable references cr_1, cr_2..., but here are two cases:

1. Exists only one callable reference with given receiver and function name.
	- In this case we have to do with such callable reference as usual argument
	
2. There are several callable references:
	- Then we have to deal with it as lambda.
	
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

*NOTE:* If receiver is callable reverence then we always have to do with it like usual argument.

For function literal arguments we constrain shape types.
Example: for lambda `{ x: Int, y ->  some code }` a shape type is \[(Int, ???) -> ??? or ???.(Int, ???) -> ???\].

*NOTE:* We can't express receiver type for lambda expression. Also we can't figure out precise function type for lambda.

> TODO: `val a = { 5 }` what type has `a`?


#### Data flow info for arguments

Data flow info for receiver must be considered for arguments. Also data flow info for first argument must be considered for second argument, e.t.c.

> TODO: for call `foo(arg2 = ..., arg1= ...)` data flow info counted in the wrong order.

Examples:
```Kotlin
fun foo(a: () -> Int, b: Int, c: () -> Int) {}

fun test(x: Int?){
	x?.plus(x) // x in argument is not null

	foo({x}, x!!, {x}) // both smart cast are correct	
}
```

As we see above, data flow info for lambda contain all data flow info for usual arguments. 
It is correct, because in runtime we calculate all arguments before running lambda(for them we just create anonymous object)

```Kotlin
class X {
	fun foo() {}
	fun foo(i: Int) {}
}
fun foo(() -> Unit, x: X) {}

fun test(x: X?) {
	foo(x!!::foo, x) // smartcast for second argument
}
```
Data flow info for receiver of callable reference must be considered for following arguments even callable reverence have several candidates.




















