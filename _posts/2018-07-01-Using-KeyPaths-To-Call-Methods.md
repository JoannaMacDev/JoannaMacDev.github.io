---
layout: post
title: Using KeyPaths to Call Methods
---

### Introduction

Swift uses the concept of KeyPaths to allow access to properties on types like `struct` and `class`; or, to be more accurate, access to stored vars, since computed properties or vars are not included.

So, being of an inquisitive and (some may say) devious nature, I thought, since computed vars are essentially methods, I would investigate whether it were possible too work out how to use KeyPaths to access methods as well.

### Warning!

Things get ugly 🤓

### An Example Class

```
class Person
{
  var name: String
  
  var age: Int
  
  private var data: String?
  
  init(name: String, age: Int, data: String? = nil)
  {
    self.name = name
    
    self.age = age
    
    self.data = data
  }
  
  func setData(_ data: String)
  {
    self.data = data
  }
}
```

In this somewhat contrived example, for some reason, we want to be able to set the private `data` property from the initialiser or from the `setData` method but we do not want to be able to read it externally. I did say contrived didn't I?

Now, if we try to use KeyPaths to access the name and age properties, we will have no problem in either getting or setting their values:

```
{
  var p = Person(name: "Steve Jobs", age: 21)
  
  p[keyPath: \Person.name] = "Tim Cook"
  
  print(p[keyPath: \Person.name])
}
```

But it is not possible to call the `setData` method with a KeyPath. So, let's create a property that points to a closure, in the hope that we can fix that.

### Closures to the Rescue!

Here's the same class but with a lazy var that holds a closure instead of a method.

```
class Person
{
  var name: String
  
  var age: Int
  
  private var data: String?
  
  init(name: String, age: Int, data: String? = nil)
  {
    self.name = name
    
    self.age = age
    
    self.data = data
  }

  lazy var setData: ((String) -> ()) =
  {
    self.data = $0
  }
}
```

We need to use a `lazy var` because, with either a `let` or a `var`, `self` would not be available from within the closure as it has not yet been fully instantiated.

Which then results in the following fairly easily readable, code:

```
{
  var p = Person(name: "Steve Jobs", age: 21)
  
  p[keyPath: \Person.setData]("MacBook Pro")
}
```

### Can We Do the Same Thing with Structs?

Well, yes, but this is where, as promised, it gets ever so slightly ugly.

If we try to use the same closure in a `lazy var`, we end up with a compiler error:

```
struct Person
{
  var name: String
  
  var age: Int
  
  private var data: String?
  
  init(name: String, age: Int, data: String? = nil)
  {
    self.name = name
    
    self.age = age
    
    self.data = data
  }

  lazy var setData: ((String) -> ()) =
  {
    self.data = $0 // ERROR - Closure cannot implicitly capture a mutating self parameter
  }
}

```

Logically this is only to be expected, as we would normally need a mutating method to alter the state of a struct; but does it mean that we cannot achieve the same functionality with a struct?

### Attempt 1

Let's try adding a mutating function and then call that from our closure:

```
struct Person
{
  …
  
  private mutating func mutatingSetData(_ data: String)
  {
    self.data = data
  }
  
  let setData: ((Person) -> (String) -> ()) =
  {
    sender in
    
    return {
             sender.mutatingSetData($0) // ERROR - Cannot use mutating member on immutable value: 'sender' is a 'let' constant
           }
  }
}
```

To start with, the closure signature might seem a bit odd at first glance but we have to remember that all methods on classes and structs take an implicit `self` as their first parameter. Thus we end up with, what seems to be, a closure that receives the "sender" and which then returns the inner closure to set the value.

Unfortunately it would seem that "input" parameters to a closure are implicitly constant and, thus cannot be used to call a mutating function.

### Attempt 2

Let's change the "sender" parameter to `inout`

```
struct Person
{
  …
  
  private mutating func mutatingSetData(_ data: String)
  {
    self.data = data
  }
  
  let setData: ((inout Person) -> (String) -> ()) =
  {
    sender in
    
    return {
             sender.mutatingSetData($0) // ERROR - Escaping closures can only capture inout parameters explicitly by value
           }
  }
}
```

This time, a different error message but, essentially the same problem.

I tried to see if adding `@noescape` to the closure would make any difference but got another error: `"@noescape attribute only applies to function types"`

### Attempt (more than 3)

Eventually all this messing around had taught me something; I now knew the correct closure signature required to assign a mutating method to my closure var. So we end up with:

```
struct Person
{
  …
  
  private mutating func mutatingSetData(_ data: String)
  {
    self.data = data
  }
  
  let setData: ((inout Person) -> (String) -> ()) = mutatingSetData
}
```

The calling code now becomes:

```
{
  var p = Person(name: "Steve Jobs", age: 21)
  
  p[keyPath: \Person.setData](&p)("MacBook Pro")
}
```

Slightly odd at first glance but, if you think about the implicit `self` parameter to all struct methods, it's sort of understandable.

### Summary

So there you have it; a means for calling methods on both classes and structs using KeyPaths. In a perverted sense it was fun to discover if it could be done but, was it worth the effort? Let me know what you think.
















































