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
