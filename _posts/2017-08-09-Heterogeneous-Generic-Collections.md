---
layout: post
title: Heterogeneous Generic Collections
---

### Introduction

Coming to Swift from a background of C#, there are areas of Swift's implementation of generic types that are still, in my opinion, sadly lacking. One of which is the ability to hold a heterogeneous list of items, of a generic type bound to differing type parameters, unless that list is simply an array of Any. This then entails having to test for each item's type followed by a forced cast to that type, in order to call the appropriately typed method on the item.

### Define a Generic Thing

```swift
struct Thing<typeT>
{
  var value: typeT
}
```

### Create a List of Generic Things

At the moment, there really is only one way to create a list that can contain both `Thing<String>` and `Thing<Int>` instances :
  
```swift
var items: [Any]
```

If we only wanted to be able to deal with generic classes, we could always create a non-generic base class, which would possibly be empty if there were no non-generic behaviour to implement.

But we want a mechanism that would let us work with, not only classes but, also, structs; which is where I was reminded of that famous book on Design Patterns by the "Gang of Four" and, in particular, the Visitor pattern.

### The Visitor Pattern Defined

To cut a long story short, the Visitor pattern allows us to define a common Visitable protocol, which can be implemented by **any** type to which we want to apply a common behaviour.

There are two simple protocols involved :

```swift
protocol Visitor { }

protocol Visitable
{
  func accept(visitor: Visitor)
}
```

The Visitor is an empty protocol that we use as the basis for creating further protocols that can define sets of strictly typed common behaviours that we wish to apply to a group of types.

e.g. for our generic `Thing` struct, we want to be able to pretty-print each one, according to their type. So we start by declaring a protocol that has one `visit…` method for each different specific type :

```swift
protocol ThingVisitor : Visitor
{
  func visitStringThing(_ thing: Thing<String>)
  
  func visitIntThing(_ thing: Thing<Int>)
}
```

At this stage, we have only declared a protocol that defines the group of types that we want to be able to visit. Next we need to create an implementation of that protocol to do the particular task of "pretty printing" each type appropriately :

```swift
struct PrintVisitor : ThingVisitor
{
  func visitStringThing(_ thing: Thing<String>)
  {
    print("String with value of \(thing.value)")
  }
  
  func visitIntThing(_ thing: Thing<Int>)
  {
    print("Int with value of \(thing.value)")
  }
}
```

Now that we have defined an implementation of the Visitor protocol, we now need to implement the Visitable protocol on our generic class. We start with a basic method :

```swift
extension Thing : Visitable
{
  func accept(visitor: Visitor)
  {
    …
  }
}
```

Then we need to determine exactly which derived visitor protocol we want to handle :

```swift
extension Thing : Visitable
{
  func accept(visitor: Visitor)
  {
    if let visitor = visitor as? ThingVisitor
    {
      …
    }
  }
}
```

If we could use the same code as for visiting non-generic types, then we could test for the type of `self`and simply call the appropriate `visit…` method, passing `self` as the parameter to the method :

```swift
extension Thing : Visitable
{
  func accept(visitor: Visitor)
  {
    if let visitor = visitor as? ThingVisitor
    {
      switch self
      {
        case is Thing<String>:
          visitor.visitStringThing(self)
        case is Thing<Int>:
          visitor.visitIntThing(self)
        default:
          break
      }
    }
  }
}
```

Unfortunately, due to Swift's lack of covariance, we are going to need to do a little dance.

In order to circumvent the lack of native generic covariance, we can add one method and a private initialiser to our generic type :

```swift
struct Thing<typeT>
{
  var value: typeT
  
  init(value: typeT)
  {
    self.value = value
  }
  
  private init<otherT>(other: Thing<otherT>)
  {
    self.value = other.value as! typeT
  }
  
  func covariantCast<otherT>() -> Thing<otherT>
  {
    return Thing<otherT>(other: self)
  }
}
```

The `covariantCast()` method calls a private initialiser, which simply uses a forced cast to convert the "other" value's type to the generic `typeT` parameter's type, to create a copy of the instance with the correct type parameter. This is possible because, logically, the true type of the Thing's value must always be the same type as its generic type.

Thinking about what we are doing here, having to create a copy of the Thing before calling the visitor's method on it, means that we have, in effect, captured the Thing's state at the time of the call and are unlikely to have to deal with any concurrency issues on the original Thing.

So, now, we can test on the type of the Thing's type parameter and then pass a correctly typed copy of the instance to the appropriate visitor method :


```swift
extension Thing : Visitable
{
  func accept(visitor: Visitor)
  {
    if let visitor = visitor as? PrintVisitor
    {
      switch typeT.self
      {
        case is String.Type:
          visitor.visitStringThing(self.covariantCast())
        case is Int.Type:
          visitor.visitIntThing(self.covariantCast())
        default:
          break
      }
    }
  }
}
```

### The End Result

After all this preparatory work, we can now write our code to traverse a heterogeneous list of generic Things; the only noticeable difference to our list is that, instead of being `[Any]`, it is now `[Visitable]`. 

```swift
  {
    let stringThing = Thing<String>(value: "Swift")
    
    let intThing = Thing<Int>(value: 123)
    
    let items: [Visitable] = [stringThing, intThing]
    
    let v = PrintVisitor()
    
    for item in items
    {
      item.accept(visitor: v)
    }
    
    …
  }
```

### Summary

In my opinion, Swift's generics system is still far from complete; there are still some gaping holes in what we are allowed to do.

Yes, we can use type erasure to hold lists of protocols references with associated types but that is still restricted to using `[Any]` if we want to mix associated types.

This article discussed an alternative to declaring and using "generic" protocols, allowing us to apply common, strictly-typed, behaviour to a heterogeneous list of variously typed generic structs.

Is it valid? Is it simpler? What are its limitations? I'll let you decide. Let me know your thoughts.


















































