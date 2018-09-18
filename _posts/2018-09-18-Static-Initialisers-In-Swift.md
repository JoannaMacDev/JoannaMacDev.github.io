---
layout: post
title: Static Initialisers in Swift
---

### Introduction

Sometimes we come across the need to "initialise" the type of a class or struct before creating instance of that class or struct. In my experience, that is often to register the type with a factory or other mechanism.

I have a validation framework that requires a type to register one or more validation closures, to be executed when an attempt to change the value of a property, or before an object is saved to storage.

When I first wrote the framework in C#, I was able to use a static constructor, which was called when the type was first referenced in code, to register static validation methods for the properties of that particular type.

Swift doesn't have the concept of a static constructor and so I thought i would see what I could do myself.

### Starting with a Protocol

```
public protocol StaticInitialisable
{
  static var staticInit: (() -> ()) { get }
}
```

All we are doing here is declaring that any implementing type contains a static closure that can be called on the type before actually instantiating the type.

### Implementing the protocol

This then allows us to declare an implementation on a type, something like this example of registering the type with a validation framework:

```
extension Person : StaticInitialisable
{
  public static let staticInit: (() -> ()) =
  {
    Validation.register(Person.self, keyPath: \Person.name)
    {
      owner, args, context in
      
      if args.newValue == nil
      {
        args.Message = "The name of a Person cannot be nil. Do you wish to continue?";
        
        args.ErrorSeverity = ValidationErrorSeverity.Confirm;
      }      
    }
    
    …
  }
}
```

### Calling the Static Initialiser

In the case of my framework, I have a base class that all "business" classes have to derive from, which has a parameterless initialiser, which will be called as `super.init()` from all subclasses

```
open class Object
{
  …
  
  private static var initialisedTypes = [StaticInitialisable.Type]()
  
  public required init()
  {
    if let staticInitialisableType = type(of: self) as? StaticInitialisable.Type,
       !Object.self.initialisedTypes.contains { $0 == staticInitialisableType }
    {
      Object.self.initialisedTypes.append(staticInitialisableType)
      
      staticInitialisableType.staticInit()
    }
  }
  
  …
}
```

In order to avoid calling the static initialiser more than once, we need a mechanism to record if it has already been called. In the case of the hierarchy that will derive from this class, I decided to maintain a static list of already initialised types, to check against.

The normal `init()` starts off by checking whether the subclass implements `StaticInitialisable`, then it checks to see if the type has already been statically initialised. If it has not yet been initialised, it is added to the list and the static initialiser is called. Next time the `init()` is called, because the type is already in the list, the static initialiser will not be called.

Of course, the static list could be declared globally and, instead of limiting its scope to a hierarchy of classes, it would be possible to use the same mechanism for any type imaginable.

Unfortunately, I have not been able to find an easy way to declare a default initialiser within an extension to the protocol, so it would be necessary to implement the "run once" code for every type that needed static initialisation.

### Summary

If you have need of static initialisation for your types, this article may come in useful, even if you have to modify how it is implemented. Please let me know if you think you can improve on it.
