---
layout: post
title: Observers, Events and Closures - Part 3
---

### Introduction

So far, in this series, we have looked at the basic principles of events, closures and delegates. In the first article, we covered how to create an event mechanism that allowed the addition of multiple closures to an event.

In that article, I discussed the possible use of enums to wrap closures, so that we could both add and remove closures from an event. Unfortunately, this technique has its limitations; in that, if we wanted to be able to remove closures from an event, it is only feasible to add one closure per enum case to the event. Why? Because the enums implement Equatable, which uses the hashValue of the enum case to test for equality, and the index(of:_) method on Array<T> will return the first element of a particular enum case, which may not be the one we actually wanted to remove.

In this article, as mentioned in an addendum to Part 1, we will look at using a struct to wrap closures; something that does allow us to both add and correctly remove closures from an event, but at the expense of a slightly more obtuse syntax for declaring the handling closures.

I will incude all relevant code, so that this article stands complete in itself and avoids having to refer back to previous articles.

### The Closure

To start, we define the base EventArgs protocol:

```swift
public protocol EventArgs { }

public struct EmptyArgs : EventArgs { }
```

The EventArgs protocol can be implemented by either structs or classes, to allow the passing of multiple parameters in one object.

If we use a struct, we would only be able to pass parameters to the closure as let properties. But, with a class, we can also use var properties to return values from the closure.

We also provide an empty struct, for those times when parameters are not needed in the closure.

Now we can define a simple typealias for the fundamental event closure signature:

```swift
public typealias EventClosure<senderT, argsT : EventArgs> = (senderT, argsT) -> ()
```

Then we redefine the concept of the closure wrapper; this time, not as an enum but, as a wrapper.

```swift
public class Closure<senderT, argsT : EventArgs> : Equatable
{
  let closure: EventClosure<senderT, argsT>
  
  init(closure: @escaping EventClosure<senderT, argsT>)
  {
    self.closure = closure
  }
  
  func invoke(sender: senderT, args: argsT)
  {
    closure(sender, args)
  }
  
  public static func ==(lhs: Closure, rhs: Closure) -> Bool
  {
    return lhs === rhs
  }
}
```

The first thing you should notice is that we are using a class and not a struct; for one simple reason: we can use the identical-to (===) operator to implement the equal-to (==) operator for Equatable; something not possible with value types.

Very simply, this class does three things:

  1. It takes a closure as parameter to the initialiser and holds a reference to it
  2. The invoke(sender:_args:_) method does what it says on the tin and invokes the closure
  3. It implements Equatable by using the === operator to compare two references the the class based on their identity

### The Event

```swift
public func +=<senderT, argsT : EventArgs>(_ event: Event<senderT, argsT>, closure: Closure<senderT, argsT>)
{
  event.add(closure)
}

public func -=<senderT, argsT : EventArgs>(_ event: Event<senderT, argsT>, closure: Closure<senderT, argsT>)
{
  event.remove(closure)
}

public class Event<senderT, argsT : EventArgs>
{
  // MARK: private properties
  
  private lazy var eventClosures: [Closure<senderT, argsT>] =
  {
    return [Closure<senderT, argsT>]()
  }()
  
  private let sender: senderT
  
  // MARK: - fileprivate methods
  
  fileprivate func add(_ closure: Closure<senderT, argsT>)
  {
    eventClosures.append(closure)
  }
  
  fileprivate func remove(_ closure: Closure<senderT, argsT>)
  {
    if let index = eventClosures.index(of: closure)
    {
      let _ = eventClosures.remove(at: index)
    }
  }
  
  // MARK: - contructors
  
  public init(sender: senderT)
  {
    self.sender = sender;
  }
  
  // MARK: - public methods
  
  public func invoke(with args: argsT)
  {
    DispatchQueue.global(qos: .background).async
    {
      self.eventClosures.forEach
      {
        $0.invoke(sender: self.sender, args: args)
      }
    }
  }
}
```

The Event class is little changed from previous versions. The main changes are to the generic parameters we are using, the type of closure we are managing and, for good measure, the use of GCD to move the invocation of the closures to a background thread.

In summary, the event class takes the object that holds the event as parameter to the initialiser, it holds an array of closure wrappers and its invoke(with:_) method uses a background thread to iterate through the closures, invoking them in turn.

### Making Use of the Event - the NotifyPropertyChanged Protocol

```swift
public protocol NotifyPropertyChanged
{
  var propertyChanged: PropertyChangedEvent { get }
}
```

The NotifyPropertyChanged protocol is very simple, in that it only contains the propertyChanged event. The idea is that observing objects can be notified of changes to a subject object's properties.

In order to make this compile, we need to implement the `EventArgs` protocol:

```swift
public struct PropertyChangedEventArgs : EventArgs
{
  public static let allProperties = PropertyChangedEventArgs(propertyName: nil)
  
  public let propertyName: String?
}
```

… and derive from the `Event<senderT, argsT>` class

```swift
public class PropertyChangedEvent : Event<Any, PropertyChangedEventArgs> { }
```

The propertyName property in the PropertyChangedEventArgs struct is optional, with the intent that passing nil would indicate that more than one property has changed.

We can also provide a convenience method in an extension to the NotifyPropertyChanged protocol, which will save us having to create the args object every time we want to invoke the event.

```swift
extension NotifyPropertyChanged
{
  func onPropertyChanged(_ propertyName: String)
  {
    let args = PropertyChangedEventArgs(propertyName: propertyName)
    
    propertyChanged.invoke(with: args)
  } 
}
```

### The Test Subject

```swift
public class TestSubject : NotifyPropertyChanged
{
  public var name: String = ""
  {
    didSet
    {
      onPropertyChanged("name")
    }
  }
  
  public lazy var propertyChanged: PropertyChangedEvent =
  {
    return PropertyChangedEvent(sender: self)
  }()
}
```

We have to implement the `propertyChanged` event property as a lazy var because we need to pass self as sender to the initialiser; something not possible in a let or regular var property.

All that is left is to add a didSet block to any property of which you wish to be notified of changes.

### The Test Code

Once again, to make our code simpler to read, we define a class derived from `Closure<senderT, argsT>` to save us having to write the sender and args types every time we want to use it.

```swift
public class PropertyChangeClosure : Closure<Any, PropertyChangedEventArgs> { }
```

In the case of NotifyPropertyChanged, we can only use Any as the sender type, as the NotifyPropertyChanged protocol could be implemented by any type and we sometimes want to be able to handle property changes from more than one type in the same closure.

```swift
class TestViewController : UIViewController
{
  …
  
  public var subject = TestSubject()
  
  override func viewDidLoad()
  {
    super.viewDidLoad()
    
    subject.propertyChanged += propertyDidChange
  }
  
  lazy var propertyDidChange: PropertyChangeClosure =
  {
    return PropertyChangeClosure
    {
      [unowned self] sender, args in
      
      if args.propertyName == "name"
      {
        DispatchQueue.main.async
        {
          self.updateNameLabel()
        }
      }
    }
  }()
  
  …
}
```

If we want to be able to remove closures from events, in order to stop receiving notifications of changes, we need to be able to hold a reference to the closure.

Therefore, instead of simply declaring a method that we can wrap in a closure and add to the event, we really need to declare our closure as a lazy var (once again, because we need to pass self into it).

Because we dispatched the invocation on a background thread, we need to wrap the UI update in a closure to be executed on the main thread

Of course, it is still possible to add a method to an event by wrapping it in a PropertyChangeClosure "on the fly":

```swift
class TestViewController : UIViewController
{
  …
  
  public var subject = TestSubject()
  
  override func viewDidLoad()
  {
    super.viewDidLoad()
    
    subject.propertyChanged += PropertyChangeClosure(closure: propertyDidChange)
  }
  
  func propertyDidChange(sender: Any, args: PropertyChangedEventArgs)
  {
    if args.propertyName == "name"
    {
      DispatchQueue.main.async
      {
        self.updateNameLabel()
      }
    }
  }
  
  …
}
```

… but then we would be back to the same old problem of not having a reference to the closure in order to be able to remove it from the event; which rather defeats the whole idea of wrapping closures to add them to an event.

### Summary

After exploring other ways of implementing a system of **event and closures**, finally, we have one that allows us to both add **and** remove closures from an event. what is more, we have learnt, not just how to do it right but, also, how to do it wrong.

Although, in using the word "wrong", wrapping closures in enums is still very valid when you have several events on one subject (as in the TextField events example) where you only ever foresee having one "delegate" per subject.

Now that we have a "fully functioning" multi-subject, multi-observer NotifyPropertyChanged system up and running, we can now move on to using it in anger.

The next article will look at simplifying the setting up of a UITextField for editing tricky things, like decimal numbers.

We'll do this by providing an Interactor to "connect" the text field to a property on an object; the idea being that: keystrokes can be validated, the property will be updated as soon as it is edited in the text field, and the text field will be updated when the property is changed on the object; you could think of it like bindings from AppKit but that work with UIKit as well



















