# Kotlin name resolution

## Some definitions

Implicit receiver: ... //TODO

Scope chain: ... //todo

Explicit receiver: ...

Local function or variable: ...

## Name resolution without implicit receivers

Suppose we have next piece of code:

```kotlin
// FILE: 1.kt
package a

fun bar(number: Int) = println("number = $number")

// FILE: 2.kt
package b

fun bar(some: Any) = println("some value = $some")

// FILE: 3.kt
package c
import a.*
import b.bar

fun bar(name: String) = println("name = $name")

fun main(args: Array<String>) {
    bar("Anton") // some value = Anton

    bar(some = "Pavel") // some value = Pavel
    
    bar(name = "Mary") // name = Mary

    bar(1) // some value = 1

    bar(number = 2) // number = 2
}
```

What happens when we resolve call `bar(...)` with some parameters? 
We create scope chain which includes next scopes: 
- local scope for function main (descriptors = {`args`})
- scope with explicit imports (descriptors = {`fun bar(some: Any)`})
- scope with members from current package (`fun bar(name: String)`)
- scope with descriptors imported by `*` (`fun bar(number: Int)`)
- some additional scopes with descriptors from kotlin stdlib.
    
After this in each scope from scope chain we try to find suitable function.
Example: when we resolve `bar("Anton")` we do the following:
- for local scope we don't see any functions with name bar
- for scope with explicit imports there is function `fun bar(some: Any)` and this function is chosen. 

Another example for call `bar(number = 2)`:
- for local scope there is no descriptors with name bar
- for scope with explicit imports there is function `fun bar(some: Any)`. This function is not suitable because it doesn't have parameter with name `number`
- scope with current package members: function `fun bar(name: String)` not suitable for the same reason
- scope with descriptors imported by `*`. There is function `fun bar(number: Int)` that matches.

So what about overloads? 
For each scope there is a group of descriptors.
For each group we collect all suitable functions and if there is more than one such function, we choose most specific one if possible.
Example:
```kotlin
// FILE: 1.kt
package a

fun foo(number: Int) = println("int = $number")

// FILE: 2.kt
package b
import a.*

fun foo(a: Any) = println("any = $a")

fun main(args: Array<String>) {
    foo(2) // any = 2
}
```
Here, when we resolve call `foo(2)` we have three groups of descriptors:
- `{args}`
- `{fun foo(a: Any)}`
- `{fun foo(number: Int)}`.
We choose function `fun foo(a: Any)` because scope with this function is closer in our scope chain for call `foo(2)` than scope with function `fun foo(number: Int)`.
In other words we do not try to shose the most specific one from these two functions because they are located in different scopes.
If they both were declared in file `2.kt` then we would choose `fun foo(number: Int)` as more specific than `fun foo(a: Any)`.

Motivations:
- explicit import should hide descriptors which are imported by `*`
- there is no scope for file, because moving function to another file in the same package should not change resolution.



## Name resolution and local descriptors

Local functions and properties have higher priority than all other descriptors (even members).
Example:
```kotlin
interface A {
    val foo: Int get() = 3
}

fun main(args: Array<String>) {
    val foo = 2
    val bar = "local"
    class B: A {
        val bar = "B"
        fun test() {
            println(foo) // 2
            println(bar) // "local"
        }
    }
    B().test()
}
```

Also "more local" function has higher priority:
```kotlin
fun main(args: Array<String>) {
    fun foo() = println("f1")
    fun test() {
        fun foo() = println("f2")

        foo() // "f2"
    }

    test()
}
```

Motivation - anonymous objects:
```kotlin
interface A {
    val foo: Int
}

fun createA(foo: Int) = object : A {
    override val foo = foo
}
```


