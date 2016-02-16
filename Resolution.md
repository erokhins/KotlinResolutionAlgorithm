# Kotlin name resolution

## Some definitions

Explicit receiver: receiver which is directly specified: `a.foo()`. Here `a` is explicit receiver for function call `foo`

Implicit receiver: some `this` available in this context.
Examples of implicit receiver:
```kotlin
fun Int.foo() {
    // `this`: Int
}

fun test() {
    with("str") {
        // `this`: String    
    }
}

open class A {
    init {
        // `this`: A
        // also here avaliable implicit this for companion object:
        test() // this function resolved to function `test` from companion object
    }
    
    companion object {
        fun test()
    }
}
```

Local functions or variables: function parameters and functions and variables declared inside function body.
Examples:
```kotlin
fun test(a: Int) { // here `a` is local variable
    val b = a // `b` also local variable
    
    fun bar() {} // `bar` is local function
    
    class A {
        val e = a // `e` is class property - not local variable
        init {
            val d = a // `d` is local variable
        }
    }
}
```

## Top-level scope chain

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
In other words we do not try to choose the most specific one from these two functions because they are located in different scopes.
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


## Name resolution for call with explicit receiver
In this section we suppose that our call has explicit receiver. For example: `a.foo()`. Here `a` is explicit receiver.
Also we suppose that this call is just a call - without magic `invoke` operator(see section `Magic invoke function`).

In call `a.foo()` `foo` can be:
- member of `a`
- some extension function (local or non-local)
- member extension function.
 
Example of member extension function call:
```kotlin
class A
class B {
    fun A.foo() = println("I'm member extension function")
}

fun main(args: Array<String>) {
    with(B()) {
        val a = A()
        a.foo() // I'm member extension function
    }
}
```

We search for functions with name `foo` in this order:
- members of `a`
- local extension function(also more local functions have higher priority here)
- for each implicit receiver(immediate receiver come first) we search member extension
- all other extension function from top-level scope chain with order described in section `Top-level scope chain`.

Examples:
```kotlin
class A {
    fun foo() = println("I'm member")
}

fun main(args: Array<String>) {
    val a = A()
    a.foo() // I'm member
}
```

```kotlin
class A
 
fun main(args: Array<String>) {
    val a = A()
    
    fun A.foo() = println("I'm local fun")
    
    fun test() {
        fun A.foo() = println("I'm more local fun")
        
        a.foo() // I'm more local fun
    }
    test()
    
    a.foo() // I'm local fun
}
```

```kotlin
class A
class B {
    fun A.foo() = println("I'm member extension from class B")
}

class C {
    fun A.foo() = println("I'm member extension from class C")
}

fun main(args: Array<String>) {
    val a = A()
    with(B()) {
        a.foo() // I'm member extension function from class B
        
        with(C()) {
            a.foo() // I'm member extension function from class C
        }
    }
}
```

```kotlin
// FILE: 1.kt
package a

class A
fun A.foo() = println("foo from package a")

// FILE: 2.kt
package b
import a.*

fun A.foo(a: Any) = println("foo from package b")

fun main(args: Array<String>) {
    val a = A() 
    a.foo() // foo from package b
}
```

## Name resolution for call without explicit receiver
As in previous part we suppose that there are no magic `invoke` functions. Also we suppose that there are no static functions with name `foo`.
See sections `Static functions from java` and `Magic invoke function`.

For call `foo()` we do the following:
- search for local function with name `foo` (more local function win)
- for each implicit receiver available in this context we try to resolve call `foo()` like `this.foo()` where `this` is some implicit receiver.
- all other non-extension function from top-level scope chain with order described in section `Top-level scope chain`.

Examples:
```kotlin
class A {
    fun foo() = println("member of A")
}

fun main(args: Array<String>) {
    fun foo() = println("local fun")
    
    with(A()) {
        foo() // local fun
    }
}
```

```kotlin
class A {
    fun foo() = println("member of A")
}

class B
fun B.foo() = println("extension for B")

fun main(args: Array<String>) {
    with(A()) {
        with(B()) {
            foo() // extension for B
        }
    }
}
```

Motivation: see discussion on https://youtrack.jetbrains.com/issue/KT-10510

## Static functions from java
Static function are handled in a special way.
Suppose static function `foo` is declared in some class `A`. 
This static function will be available as `foo()` only in some class which is inherited from A (maybe not directly).
Suppose this class has name `B`. Then we consider this static function right after we try `this@B.foo()`.
  
Example:
```kotlin
// FILE: A.java
public class A {
    public static void foo() {
        System.out.println("static foo");
    }
}

// FILE: 1.kt
class C {
    fun foo() = println("member of B")
    
    inner class B : A() {
        init {
            foo() // "static foo" like in java            
        }
    }
}

fun main(args: Array<String>) {
    C().B()
}
```

## Magic invoke function
As you know from previous sections call `a.foo()` can mean several things: member call, extension call, etc.
But in all considered cases `foo` was a function, which is not always true
Sometimes, we may have property `foo` which is has type `() -> Int`. In this case `a.foo()` is almost the same as `a.foo.invoke()`.
But `foo` also can have type `A.() -> Int`. In this case this call is equal to this: `foo.invoke(a)`.

Note: `foo` can be some extension property for some implicit receiver or even more - some member extension property:
```kotlin
class A {
    val B.foo: C.() -> Unit get() = { println("Hello") }
}
class B
class C

fun main(args: Array<String>) {
    val c = C()
    with(A()) {
        with(B()) {
            c.foo() // Hello
            
            with(c) {
                // this call has 3 implicit receivers: A, B, C. Two of them (A, B) used for property `foo`.
                foo() // Hello
            }
        }
    }
}
```

What about priority of such functions? It is pure magic!
