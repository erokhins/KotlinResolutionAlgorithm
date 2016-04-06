Crazy examples:

```kotlin
fun ((Int) -> Unit).test() {}

fun test() {
    { println(it) }.test()
}
```
