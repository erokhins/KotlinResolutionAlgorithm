# KotlinResolveAlgorithm
Description for new resolve algorithm

### ExpressionContext
This context contains:
- Scope
- Expected type
- Resolve mode(ONLY_RESOLUTION or RESOLUTION_WITH_INFERENCE)

### Expression Typing Facade
For each expression with ExpressionContext we can get JetTypeInfo.
If Resolve mode was RESOLUTION_WITH_INFERENCE then JetTypeInfo contais that information:
- JetType for expression
- data flow info

Otherwise, when resolve mode is ONLY_RESOLUTION, JetTypeInfo contains:
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
For `test()` expression we created context with NO_EXPECTED_TYPE and RESOLUTION_WITH_INFERENCE, because it is top-level call
Result was: JetType = Int, empty data flow info.

```Kotlin
fun emptyList<E>(): List<E>

val list: List<Int> = emptyList()
```
For `emptyList()` context have expected type `List<Int>` and same resolve mode: RESOLUTION_WITH_INFERENCE.
But we always do resolve with ONLY_RESOLUTION, after that if real resolve mode was RESOLUTION_WITH_INFERENCE
we complete inference i.e. solve constrain system.

Result for first stage:  
- Constrain system: 
	- Type variable: `E`
	- return type `List<E>`
	- E <: Int


> TODO: Remove addition data flow info after resolve function body

### Call Resolver

*Input:*
- call `[r].foo(a_1, a_2, ..., fl_1, fl_2, ... cr_1, cr_2 , ...)`
	- fl_1, fl_2... are function literal arguments;
	- cr_1, cr_2... - callable references
	- order of arguments can be any
- CallContext - ExpressionContext with some addition info

