---
layout: post
title: Observers, Events and Closures
---

### Introduction

In my last post ([Delegates vs Closures in Swift](https://joannamacdev.github.io/Delegates-And-Closures/)) I mentioned that there were problems in removing closures from arrays. This post is the first in a small series, describing an implmentation of the **Observer Pattern** as a system of events and closures. It will also cover an interesting approach to implementing delegates.

Instead of relying on that previous article as a starting point, I intend to start again from scratch, as there are several changes, even to the most fundamental of types and associated code.

#### Closures

Whereas, in the previous article, closure types were simply type aliases for closure signatures, due to the inability to successfully remove them from arrays, we needed to come up with a solution that would allow correct handling in arrays.

Two possibilities came to mind:

  The first was to use a wrapper struct or class but that overcomplicated the syntax for the user code

  The second was (possibly) surprising; [an enum with an associated value](#enums-for-closures) for the closure.

##### A Base Args Type

In the previous article, we used a base EventArgs class and inherited other args classes from that but, in the spirit of **protocol oriented programming** that Swift encourages, we now have an empty protocol, which can be implemented by either a class or a struct.

```swift
public protocol EventArgs { }

public struct EmptyArgs : EventArgs { }
```

Then, for those closures which don't require any parameters, we have a simple empty struct that implements the EventArgs protocol.

##### A Base Closure Protocol

In order to store a list of [closure enums](#enums-for-closures), which can all have differing closure signatures, we are going to need a common type:

  1. for the benefit of the array
  2. so that we can invoke the closure generically

```swift
public protocol Closure : Hashable
{
  associatedtype SenderType
  
  func invoke<argsT : EventArgs>(sender: SenderType, args: argsT)
}

extension Closure
{
  public static func ==(lhs: Self, rhs: Self) -> Bool
  {
    return lhs.hashValue == rhs.hashValue
  }
}
```

Because our enum types are being stored in arrays, they will have to conform to the Equatable protocol; but I have decided to inherit our Closure protocol from Hashable, which will also give us the ability to manage them in dictionaries, should we so wish.

The `==` static method for Equatable can be implemented by default in an extension the the Closure protocol, since it only needs to access the `hashValue` tha is part of the Hashable protocol.

For any given closure, for a single event type, the closure's sender will always be of the same type as the event's sender; therefore, we can use an associated type for the sender in the `invoke(…)` method.

However, the type of the args object that we will pass to the closure can vary, according to the number and type of parameters we want to pass to the handling closure; this means we need to make this method generic, adding a type parameter for the args type that conforms to the [base args type](#a-base-args-type).

##### Enums for Closures

The idea of using an enum to hold an associated value of a closure might sound unusal, so I am starting with the real world example of a closure for use with the NotifyPropertyChanged protocol we discussed in the previous article.

We are going to need the EventClosure typealias for our bare closure type:

```swift
public typealias EventClosure<senderT, argsT : EventArgs> = (senderT, argsT) -> ()
```

… and the PropertyChangedEventArgs type, which is now a struct:

```swift
public struct PropertyChangedEventArgs : EventArgs
{
  public static let allProperties = PropertyChangedEventArgs(propertyName: nil)
  
  public let propertyName: String?
}
```

Let's take this one step at a time:

```swift
public enum PropertyChangeClosureType<senderT> : Closure
{
  case didChange(EventClosure<senderT, PropertyChangedEventArgs>)
  
```

The essence of this enum is the list of cases; in this simple example, there is only one, `didChange`.

The senderT type parameter is necessary for this closure type, because the `NotifyPropertyChanged` protocol can be implemented by virtuall any type.

We declare the `didChange` case with an associated type that reflects the closure signature associated with the `propertyChanged` event in the `NotifyPropertyChanged` protocol.
  
```swift
  
  public func invoke<argsT : EventArgs>(sender: senderT, args: argsT)
  {
    switch self
    {
      case .didChange(let closure):
        closure(sender, args as! PropertyChangedEventArgs)
    }
  }
}
```

The `invoke(…)`method in this example might seem a bit overly complicated, with a switch statement for a single case, but, for when there are more cases, I wanted to demonstrate clearly the principle of extracting the closure from the associated value and how to call it.

Since the args parameter is passed in as a generic argsT type, we need to explicitly cast it to the correct type for the closure associated with this case.

```swift
extension TextFieldEventClosureType : Hashable
{
  public var hashValue: Int
  {
    switch self
    {
      case .didChange(_):
        return 0
    }
  }
}
```

**Note:** Swift doesn't allow enums to be "RawRepresentable" and to have also have associated values, which is why we have had to implement Hashable by returning an Int value for each of the cases. 

To give you some idea of how a more complex enum would be implemented, here is one designed to fulfil the needs of an imitation of the UITextFieldDelegate:

```swift
public enum TextFieldEventClosureType : Closure
{
  case shouldBeginEditing(TextFieldPermissionClosure)
  case didBeginEditing(EventClosure<TextField, EventArgs>)
  case shouldEndEditing(TextFieldPermissionClosure)
  case didEndEditing(TextFieldDidEndEditingClosure)
  case shouldChangeCharacters(TextFieldShouldChangeCharactersClosure)
  case shouldClear(TextFieldPermissionClosure)
  case shouldReturn(TextFieldPermissionClosure)
  
  public func invoke<argsT : EventArgs>(sender: TextField, args: argsT)
  {
    switch self
    {
      case .shouldBeginEditing(let closure):
        closure(sender, args as! PermissionArgs)
      case .didBeginEditing(let closure):
        closure(sender, args)
      case .shouldEndEditing(let closure):
        closure(sender, args as! PermissionArgs)
      case .didEndEditing(let closure):
        closure(sender, args as! TextFieldDidEndEditingArgs)
      case .shouldChangeCharacters(let closure):
        closure(sender, args as! TextFieldShouldChangeCharactersArgs)
      case .shouldClear(let closure):
        closure(sender, args as! PermissionArgs)
      case .shouldReturn(let closure):
        closure(sender, args as! PermissionArgs)
    }
  }
}

extension TextFieldEventClosureType : Hashable
{
  public var hashValue: Int
  {
    switch self
    {
    case .shouldBeginEditing(_):
        return 0
      case .didBeginEditing(_):
        return 1
      case .shouldEndEditing(_):
        return 2
      case .didEndEditing(_):
        return 3
      case .shouldChangeCharacters(_):
        return 4
      case .shouldClear(_):
        return 5
      case .shouldReturn(_):
        return 6
    }
  }
}
```

These enums are used in attaching a method to an event in the following manner:

```swift
    testSubject.propertyChanged += .didChange(testObject.propertyDidChange)
```

… or, using a closure, like this:

```swift
    testSubject.propertyChanged += .didChange
    {
      sender, args in
      
      guard let propertyName = args.propertyName else
      {
        // more than one property has changed
        
        return
      }
      
      if propertyName == "name"
      {
        print("\"\(sender.name)\" has been assigned to \(propertyName)")
      }
    }
```

##### The Event Class

Now that we are using enums with associated closures, we need to change the Event class to accomodate this.

```swift
public func +=<closureT : Closure, argsT : EventArgs>(_ event: Event<closureT, argsT>, closure: closureT)
{
  event.add(closure)
}

public func -=<closureT : Closure, argsT : EventArgs>(_ event: Event<closureT, argsT>, closure: closureT)
{
  event.remove(closure)
}


public class Event<closureT : Closure, argsT : EventArgs>
{
  // MARK: private properties
  
  private lazy var eventClosures: [closureT] =
  {
    return [closureT]()
  }()
  
  private var sender: closureT.SenderType
  
  // MARK: - fileprivate methods
  
  fileprivate func add(_ closure: closureT)
  {
    eventClosures.append(closure)
  }
  
  fileprivate func remove(_ closure: closureT)
  {
    if let index = eventClosures.index(of: closure)
    {
      let _ = eventClosures.remove(at: index)
    }
  }
  
  // MARK: - contructors
  
  public init(sender: closureT.SenderType)
  {
    self.sender = sender;
  }
  
  // MARK: - public methods
  
  public func invoke(with args: argsT)
  {
    eventClosures.forEach
    {
      $0.invoke(sender: sender, args: args)
    }
  }
}
```

The Event type still has two generic type parameters but, this time, we use the closureT to determine the sender's type; the argsT type is still required for the `invoke(…)`method and it is not part of the closureT type.


























