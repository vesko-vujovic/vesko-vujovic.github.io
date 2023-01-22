---
layout: post
title:  "Python: Class Variables vs Instance Variables"
tags: [python, effective python]
---

![python](/assets/img/python_lang.jpeg)


# Introduction

In this blog post, we will talk about the distinction between class and instance variables and some of the common pitfalls, that are often made especially when you are a new Python developer. 


# Class Variables

They are declared inside the class definition they don't belong to any instance methods like `__init__` or any other method. They store their data inside the class itself, and whenever you create a new object from this class it will contain the same shared set of class variables.  This will imply that when you change the class variable the change will propagate to all object instances at the same time.

Let's see and example of this type of variable.

```python

class Guitar:

    
    no_of_strings = 1 # class variable 
    
    
    def __init__(self, brand_name: str, type_of_guitar: str):
    	# Instance variables
        self.brand_name = brand_name 
        self.type_of_guitar = type_of_guitar

```

If you want to access __no_of_strings__ you will do it like this:

``` python
Guitar.no_of_strings
```

This proves our statement that the data is stored in the class itself and not in the instantiated object. This is really important to remember.



# Instance Variables

Instance variables as the name implies are bound to object instance.The main difference from class variables is that all data is stored in each individual object created from the class. So that would also mean that when you modify an instance variable it only affect the object on which you are making the change.

If you try to access the instance variable through the class without creating an object you will get an AttributeError. 

```python
Guitar.brand_name


AttributeError: type object 'Guitar' has no attribute 'brand_name'
```

# Common pitfalls

Let's explore int his sections some of the common mistakes.
For testing purpose we will create few objects. 

```python
first_guitar  = Guitar("gibson", "electric")
second_guitar = Guitar("yamaha", "acoustic")



second_guitar.no_of_strings, first_guitar.no_of_strings

# (1, 1)

```
If we modify now `no_of_strings` in the class we would expect that it will not be reflected in other classes right that we already instantiated? Well wrong :bomb: See what happens below.

```python
Guitar.no_of_strings = 10
second_guitar.no_of_strings, first_guitar.no_of_strings

# (10, 10)
```

This didn't work because modifying the class variable on the class namespace will affect all instances of the class.

But what about changing the class variable on the specific object instance itself? :thinking:

```python
first_guitar.no_of_strings = 2
second_guitar.no_of_strings, first_guitar.no_of_strings
```

Superficially looking we could say "OK that solved our problem" if we scratch beneath the surface we will find that we jumped inside another pit. What happened in the background is that we created the __no_of_strings__ instance variable that will shadow the class variable. Let's checkout.

``` python
first_guitar.__dict__
{'brand_name': 'gibson', 'type_of_guitar': 'electric', 'no_of_strings': 2}



first_guitar.__class__.no_of_strings , first_guitar.no_of_strings
(10, 2)
```
When the variable name is inside `__dict__` that means that we created an instance variable. From the example, above you can see that __class variable__ and __instance variable__ are out of sync. The old value class variable is 10, and the new modified value is 2. This is maybe the thing that you wanted that is not necessarily bad, but it's good the be mindful that in the background you will have shadowed class variable.


To finish off let's see another interesting example actually one really useful implementation of class variables and the example on how we can mess things up if we are not mindful of what we are doing.


```python
class Car:
    
    total_manufactured = 0
    
    
    def __init__(self):
        self.__class__.total_manufactured += 1
    

Car.total_manufactured
# 0


Car().total_manufactured
# 1

Car().total_manufactured
# 2

Car().total_manufactured
# 3

Car().total_manufactured
# 4

Car.total_manufactured
# 4

```
Class variables implemented like this could be very useful whatever you want to track. In this example we see that every time we created new instance we got back incremented `total_manufactured` class variable. 

If we implemented Car class like this:

```
class Car:
    
    total_manufactured = 0
    
    
    def __init__(self):
        self.total_manufactured += 1


Car.total_manufactured
# 0

Car().total_manufactured
# 1


Car().total_manufactured
# 1


Car().total_manufactured
# 1

Car().total_manufactured
# 0
```

As you can see it's easy to make a mistake and shadow the class variable. In this case we didn't got back an increment and `class variable` and `instance variable` got out of sync.


# Final Thoughts

We have learned to differentiate between class variables and instance variables, and we saw that class variables can be a good way to reflect the changes across all instantiated objects if that's our intention, and also a good way to access the variable value without making an object instance. We also saw how it's easy to make a mistake and shadow the class variable if we don't understand what we are really doing.

Until the next blog post... :wave:










