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


