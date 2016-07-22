Crazy examples:

```kotlin
fun ((Int) -> Unit).test() {}

fun test() {
    { println(it) }.test()
}
```


All arguments should be analyzed *before* call resolver. They will be computed via transfer order => we know DataFlowInfo for arguments without name resolution.
Note: body of lambda argument will be computed inside function body => correct DataFlowInfo for body is DataFlowInfo after all arguments.

Also we should check some other cases before name resolution:
- spread before lambda or callable reference
- several external lambda arguments
 

Type argument placeholder:
```kotlin
fun <A, B, C> foo(a: A, c: C): B = TODO()

fun test() {
  foo<_, Int, _>(1, "")
}
```

Same for parameters: https://jetbrains.slack.com/archives/kotlin-design/p1459961727000398

Save information into trace for:
- named argument
- resolved call & variable in invoke call
- All staff in NewResoltuionOldInference


DataFlowInfo for arguments: https://jetbrains.slack.com/archives/kotlin-design/p1460380323000456
Another example:
```
class X {
    fun bar(x: X) {}
    
    val foo: (Any) -> Unit = {}
}

fun test(x: X?) {
    x.foo(x!!) // incorrect also

    x.bar(x!!) // incorrect
}
```
Current decision -- leave as is.



Intersection types sources:
- smart cast
- `T <: Some_1`, `T <: Some_2`
- java: `void <T> @NotNull T foo()` -- real type is `T & Any`


Star projections: https://jetbrains.slack.com/archives/kotlin-design/p1460667989000618


### Smart cast vs inference
```
class Bar<T>(val t: T)

@Suppress("TYPE_PARAMETER_OF_PROPERTY_NOT_USED_IN_RECEIVER")
val <T: CharSequence?> bar: T = TODO()

fun <T : Any> Bar<T?>.foo(vararg a: T) {}

fun test() {
    bar.foo(bar.t!!, bar.t, "") // on bar.t here may be unstable smart cast
}
```
Resolution: hard to implement, rare case.
