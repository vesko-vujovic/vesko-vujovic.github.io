---
layout: post
title:  "Python: __dict__ vs __slots__"
tags: [python, programming, effective python]
---



# Introduction

In this blog post, I wanted to talk about one really cool feature of python that every programmer should know, especially if you want to write performant code.


# \__dict__  as a default way of thinking

First, let's create these two classes for the sake of the example.e

```python
class Comment:
    def __init__(self, text, author, date):
        self.text = text
        self.author = author
        self.date = date

class Post:
    def __init__(self, name, date, author, comments):
        self.name = name
        self.date = date
        self.author = author
        self.comments = comments


comments = []

for i in range(0, 10):
    comments.append(Comment('Comment text {i}','Author{i}', "2022-12-15"))
    

first_post = Post("First post about dict vs slots", "2022-12-15", 
                  "Vesko", comments)
```

When we execute this code you will notice that every attribute and its value will be saved in the \__dict__ attribute of the class.
```python
first_post.__dict__


{'name': 'First post about dict vs slots',
 'date': '2022-12-15',
 'author': 'Vesko',
 'comments': [<__main__.Comment at 0x7fd79360f520>,
  <__main__.Comment at 0x7fd793614370>,
  <__main__.Comment at 0x7fd7936142b0>,
  <__main__.Comment at 0x7fd793614430>,
  <__main__.Comment at 0x7fd7936140a0>,
  <__main__.Comment at 0x7fd793614280>,
  <__main__.Comment at 0x7fd793614310>,
  <__main__.Comment at 0x7fd793614190>,
  <__main__.Comment at 0x7fd7936145b0>,
  <__main__.Comment at 0x7fd7936142e0>]}

```
Ok so far so good. :smiley: Can you imagine now storing millions of objects as key-value pairs? When you have a situation like this  `__dict__` will eat a lot of RAM memory.

So It's the right time to shift our thoughts towards `__slots__` and see what he can offer us.


# \__slots__ as a revelation

What are __slots__? From Python documentation we get the following definition:

> __slots__ allows us to explicitly declare data members (like properties) and deny the creation of __dict__ and __weakref__ (unless explicitly declared in __slots__ or available in a parent.)

Ok now we know can prevent the creation of __dict__ hash map and save some memory. Let's dive into the details of how it can be done and what are the benefits and downsides of this approach.


Let's create an example with `__slots__`.

```python
class PostUsingSlots:
    
    __slots__ = ["date", "name", "author", "comments"]
    
    def __init__(self, name, date, author, comments):
        self.name = name
        self.date = date
        self.author = author
        self.comments = comments


post_using_slots = PostUsingSlots("First post about dict vs slots", "2022-12-15", 
                  "Vesko", comments)
```

After we run the command on the newly created object we get:
```python
print(PostUsingSlots.__dict__)
{'__module__': '__main__', '__slots__': ['date', 'name', 'author', 'comments'], '__init__': <function PostUsingSlots.__init__ at 0x7fd7938c84c0>, 'author': <member 'author' of 'PostUsingSlots' objects>, 'comments': <member 'comments' of 'PostUsingSlots' objects>, 'date': <member 'date' of 'PostUsingSlots' objects>, 'name': <member 'name' of 'PostUsingSlots' objects>, '__doc__': None}
```

```python
print(PostUsingSlots.comments.__class__)
'<class "member_descriptor'
```
We see a lot of member descriptors :thinking:.I

## What the heck are now member descriptors :bulb:?

Before advancing further we must know the default behavior of accessing Python attributes. When we call `first_post.author` Python first calls the `__getattribute__()` method from there it looks up in `__dict__` and returns the value.  

When we define the class with `__slots__` it will replace instance dictionaries with a fixed-length array of slot values. That means that `__slots__` will create descriptor methods for each attribute and implement `__get__`, `__set__` and `__delete__` methods. `__get__`, and `__set__` will use now array instead of a dictionary that are implemented in __C__ which implies that they are very efficient.


Ok so enough talk, let's make some useful benchmarks.

## Attribute access and object creation


## Memory footprint













