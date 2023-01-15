---
layout: post
title:  "Python: __dict__ vs __slots__"
tags: [python, programming, effective python]
---

![spark](/assets/img/dict_vs_slots.avif)

# Introduction

In this blog post, I wanted to talk about one really cool feature of python that every programmer should know, especially if you want to write performant code.


# \__dict__  as a default way of thinking

First, let's create these two classes for the sake of the example.

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

class CommentUsingSlots:
    
    __slots__ = ["text", "author", "date"]
    
    def __init__(self, text, author, date):
        self.text = text
        self.author = author
        self.date = date

comments_with_slots = []

for i in range(0, 10):
    comments_with_slots.append(CommentUsingSlots('Comment text {i}','Author{i}', "2022-12-15"))



post_using_slots = PostUsingSlots("First post about dict vs slots", "2022-12-15", 
                  "Vesko", comments_with_slots)
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
We see a lot of member descriptors :thinking: :question:

## What the heck are now member descriptors :bulb:?

Before advancing further we must know the default behavior of accessing Python attributes. When we call `first_post.author` Python first calls the `__getattribute__()` method from there it looks up in `__dict__` and returns the value.  

When we define the class with `__slots__` it will replace instance dictionaries with a fixed-length array of slot values. That means that `__slots__` will create descriptor methods for each attribute and implement `__get__`, `__set__` and `__delete__` methods. `__get__`, and `__set__` will use now array instead of a dictionary that are implemented in __C__ which implies that they are very efficient.


Ok so enough talk, let's make some useful benchmarks. 

## Attribute access and object creation

Here is the snippet for our benchmark.

``` python
from timer import timer
import logging

logging.basicConfig(level=logging.DEBUG)
timer.set_level(logging.DEBUG)

@timer("Object creation", "s")
def create_post_with_comments(post_cls, cmnt_cls, post_size ,comment_size):
    for _ in range(post_size):
        
        comments = []
        
        for comment in range(comment_size):
            comments.append(cmnt_cls('Comment text {comment}','Author{comment}', "2022-12-15"))
            
        
        
        post = post_cls("First post about dict vs slots", "2022-12-15", 
                  "Vesko", comments)
        
    return post

@timer("Accessing attribute", "s")
def acces_attribute_of(cls, no_of_times):
    
    for _ in range(no_of_times):
        cls.comments

```
### Object creation
``` python
post_with_slots = create_post_with_comments(PostUsingSlots, CommentUsingSlots, 1000000, 100)

DEBUG:timer.Object creation:start
DEBUG:timer.Object creation:cost 31.309 s


ordinary_post = create_post_with_comments(Post, Comment, 1000000, 100)
DEBUG:timer.Object creation:start
DEBUG:timer.Object creation:cost 32.746 s

```
On object creation we don't get to much **~4.55%**

### Attribute access
``` python
acces_attribute_of(post_with_slots, 1000000)

DEBUG:timer.Accessing attribute:start
DEBUG:timer.Accessing attribute:cost 0.060 s



acces_attribute_of(ordinary_post, 1000000)

DEBUG:timer.Accessing attribute:start
DEBUG:timer.Accessing attribute:cost 0.067 s
```

On attribute acccess we get __~11.66__ of improvment. 




## Memory footprint

Ok, now we want to see how big footprint `__slots__` have on the memory compared to `__dict__`.

Just a small tip for this one, we will not use sys.getsizeof function because it doesn't include the size of the referenced object. So for our little experiment, we will use a library called [pympler](https://pympler.readthedocs.io/en/latest/]).

First, let's prove our statement on the behavior of the `getsizeof` function, and then we will move on to the real memory snapshot using pympler.

``` python
sys.getsizeof(ordinary_post)
# 48


sys.getsizeof(post_with_slots)
# 64
```

``` python
asizeof.asizeof(ordinary_post)
# 16896

asizeof.asizeof(post_with_slots)
# 6920
```

:scream: WOW **144.14%** of improvement compared to class that has `__dict__` :exclamation::exclamation::exclamation:

I think that with this example we concluded that `__slots__` are really saving memory big time! 


# Bad sides of `__slots__`?

While this python feature is awesome when you have a case with lots of objects and can reduce a big amount of memory it comes with a price of being more rigid regarding object attributes.

``` python
post = PostUsingSlots("First post about dict vs slots", "2022-12-15", 
                  "Vesko", comments)


---------------------------------------------------------------------------
AttributeError                            Traceback (most recent call last)
Input In [102], in <cell line: 1>()
----> 1 post.editor = "Vesko 2"

AttributeError: 'PostUsingSlots' object has no attribute 'editor'
```

Once you define `__slots__` it's set in stone and you can't add new attributes.

But... There is always but.

In situations where you want to reduce the memory footprint and at the same time have the flexibility of `__dict__` you can combine both approaches.

```python

class PostUsingSlotsAndDict:
    
    __slots__ = ["date", "name", "author", "comments", "__dict__"]
    
    def __init__(self, name, date, author, comments):
        self.name = name
        self.date = date
        self.author = author
        self.comments = comments



post = PostUsingSlotsAndDict("First post about dict vs slots", "2022-12-15", 
                  "Vesko", comments)

post.editor = "Vesko 2"
post.__dict__
{'editor': 'Vesko 2'}

```
By adding `__dict__` at the end of the list you are adding the needed flexibility to your class.

### What about inheritance?

Well, all attributes from the base class are accessible to the child class.55

```python

class PostBase:
    
     __slots__ = ["date", "name"]
    
        
class PostExtended(PostBase):
    
     __slots__ = ["editor", "author"]


post_extended = PostExtended()
post_extended.date = "2022-12-15"
post_extended.editor = "Vesko"

```

### Dataclasses 

You can use `__slots__ ` also in your Data Classes like this :point_down:

```python

from dataclasses import dataclass

@dataclass(slots=True)
class Post:
     name: str
     date: str
     author: str
     comments: list
```
# Final thougts

I hope that we now know and fully understand why we need `__slots__`  sometimes as a mechanism for reducing memory pressure.

*To summarize everything:*

**`Pros:`** *If you have a large number of objects and you need to reduce the memory pressure, `__slots__` can be seen as a very useful weapon. Given the fact that you can add `__dict__` inside the slots list gives you the advantage of both worlds, reduced memory, and dynamic attributes.* 


**`Cons:`** *You need to know in advance what class attributes need to be marked as slot members. You cannot add dynamic attributes it's a little bit rigid, but it can be mitigated with the above-mentioned trick i.e adding `__dict__` to the slots array.* 


I hope you enjoy this article! Leave your comments below if you have any thoughts. 

Until the next post. :wave:































