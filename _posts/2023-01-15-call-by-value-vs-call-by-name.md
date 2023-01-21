---
layout: post
title:  "Scala: Understanding Call By Value vs Call By Name "
tags: [scala, functional programming]
---

![spark](/assets/img/scala.png)

# Introduction

In this blog post, we will cover an important topic related to the Scala programming language i.e. evaluation strategies of call-by-value and call-by-name. This is an important topic and is sometimes overlooked by programmers.


# Call by Value

This method is the default one that Scala uses to evaluate expressions. Whenever it encounters the expression it will be evaluated right away. This is the same strategy used when you define `val`: the expression that you assign to the variable is evaluated immediately there is no deferred evaluation to evaluate everything once you need it. When the expression is passed to a function the expression itself will be evaluated first and then the function call will happen.  Let's explain everything with one example.

```scala
def addTwo(x: Int, y: Int) = x + x

addTwo(4*4, 2-1) // first function call
addTwo(16, 2-1) // first reduction of (4*4) to 16 
addTwo(16, 1)  // second reduction (2-1) to 1
16 + 16       // third reduction
32           // final reduction

```
Pay close attention to the body of the `addTwo` function. As you can see we are passing `y` which is never used inside the body of the function `x+x`.  So why the hell Scala evaluates `y` argument :question:

That's how `Call By Value` works, it will evaluate every expression whether you use it or not. Wait so that's a waist of time :thinking: . Well somehow yes, if we don't use certain argument. Here we are telling to Scala that out input argument is `Int` and return type should be also `Int`



# Call By Name

This evaluation strategy is the opposite of Call By Value strategy. The evaluation of the argument `y` is deferred until  needed.

``` scala
def add(x: Int, y: Int) = x + x


add(4*4, 2-1)   // first function call
(4*4) + (4*4)  // first reduction
16 + (4*4)    // second reduction
16 + 16      // third reduction
32          // final reduction
```
As you can see the expression `2-1` will never be evaluated. But we calculate `4*4` twice. This is normal since we have evaluation on the level of function args and then evaluation as function body. When we pass the `y` argument like this `y => Int` we are telling to Scala  to evaluate argument once it's needed.

# Stumbling stone

Ok, now that we know how the dark magic of evaluation strategy works, we should be aware of one pitfall when coding in Scala. 

```scala
def addByValue(x: Int, y: Int) = x + x
def addByNameaddByName(x: Int, y: => Int) = x + x
def iWillRunUntilEternity(): Unit = iWillRunUntilEternity

addByValue(4*4, iWillRunUntilEternity) // This function will never finish (infinite recursion)
addByName(4*4, iWillRunUntilEternity)  // This one will finish since y will never be evaluated.

```

Will never terminate because Scala will always try to evaluate `iWillRunUntilEternity` which is an infinite loop. In contrast `addByName` will finish because we don't evaluate the `y` parameter.

# Performance

Now it's time for some performance insight between these two strategies. Let's make one little experiment where we can show some stuff that are happening under the hood.

```scala
def getOne() = {
    println("calling getOne")
    1
}


def callByValue(x: Int) = {

    println("Calling callByName")
    x+x
}
def callByName(x: => Int) = {
    
    println("Calling callByName")
    x + x
}

callByValue(getOne)

//calling getOne
//Calling callByName

callByName(getOne)
//Calling callByName
//calling getOne
//calling getOne

```
You can see that `callByValue` will call only once `getOne` function whereas `callByName` will call it twice.  This is because call by value will first evaluate the expression and then call the function itself. On the opposite side call by name argument will be evaluated only inside the body of the function and when it's called. Since we call `x` two times i.e `x+x` the function will evaluate twice the argument `x`.

With this said we can conclude that `callByValue` will have better performance since it will not compute the same thing twice.

# Conclusion

In this blog post, we dived into the topic of evaluation strategies. We learned that call by value will be more efficient since it avoids re-evaluating the same expression multiple times. It's perfectly fine to use `call-by-name` strategy if you know that you are not calling the same argument more than once in the body of the function and you need evaluation to be differed until the argument is needed.

Here you can find the code used in this blog [post](https://github.com/vesko-vujovic/ScalaExamples/blob/main/CallByNameVsCalByValue.ipynb).

See you in the next blog post... :wave:















