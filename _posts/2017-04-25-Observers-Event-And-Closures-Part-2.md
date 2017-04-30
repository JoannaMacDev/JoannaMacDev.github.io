---
layout: post
title: Observers, Events and Closures - Part 2
---

### Introduction

The first article in this small series, I described an implementation of the **Observer Pattern** as a system of events and closures. The NotifyPropertyChanged protocol was relatively easy to implement and use; we could simply add a method or closure to the single event using the `+=` operator. But, the TextField event model was more complicated with seven events, some of all of which we might want to handle in a delegate object but, having to assign a closure to each of the seven events in turn can be tedious.

In this article, I intend to show how we can create a delegate mechanism that will allow us to add a single object as delegate for all those events in one line of code.

### The Event Source

We will start by defining a protocol that will allow us to add a list of closures to any type that we want to contain events.

```swift
public protocol EventSource
{
  associatedtype ClosureType : Closure
  
  func add(closures: [ClosureType])
  
  func remove(closures: [ClosureType])
}
```

We are going to use this protocol in two ways. the first is to allow us to add several closures of methods from various **observers** to the single event on objects implementing the NotifyPropertyChanged protocol.

```swift
public protocol NotifyPropertyChanged : EventSource
{
  associatedtype ClosureType : Closure
  
  var propertyChanged: Event<ClosureType, PropertyChangedEventArgs> { get }
}
```

Because the NotifyPropertyChanged protocol can be implemented by various types, it would be convenient to be able to define the `add(closure:_)` and `remove(closure:_)` methods once and for all. To achieve this, we can extend the NotifyPropertyChanged protocol with the EventSource protocol and provide a default imlementation in an extension to the NotifyPropertyChanged protocol:

```swift
extension NotifyPropertyChanged
{
  public func add(closures: [ClosureType])
  {
    closures.forEach { propertyChanged += $0 }
  }
  
  public func remove(closures: [ClosureType])
  {
    closures.forEach { propertyChanged -= $0 }
  }
}
```

As you can see, all this does is to iterate through the list of closures, adding them to the `propertyChanged` event.

### Connecting Multiple Events

The NotifyPropertyChanged protocol only has one event and, as described above, is relatively simple to implement, event though attaching multiple observers at one time is not that common.

However, the real power of the [EventSource](*the-event-source) protocol becomes apparent when we implement it on the TextField example; let's start by defining a class to imitate UITextField for our example (the code for TextFieldEventClosureType is in the first article).

```swift
public class TextField
{
  public lazy var shouldBeginEditing: Event<TextFieldEventClosureType, PermissionArgs> =
  {
    return Event<TextFieldEventClosureType, PermissionArgs>(sender: self)
  }()
  
  public lazy var didBeginEditing: Event<TextFieldEventClosureType, EmptyArgs> =
  {
    return Event<TextFieldEventClosureType, EmptyArgs>(sender: self)
  }()
  
  public lazy var shouldEndEditing: Event<TextFieldEventClosureType, PermissionArgs> =
  {
    return Event<TextFieldEventClosureType, PermissionArgs>(sender: self)
  }()
  
  public lazy var didEndEditing: Event<TextFieldEventClosureType, TextFieldDidEndEditingArgs> =
  {
    return Event<TextFieldEventClosureType, TextFieldDidEndEditingArgs>(sender: self)
  }()
  
  public lazy var shouldChangeCharacters: Event<TextFieldEventClosureType, TextFieldShouldChangeCharactersArgs> =
  {
    return Event<TextFieldEventClosureType, TextFieldShouldChangeCharactersArgs>(sender: self)
  }()
  
  public lazy var shouldClear: Event<TextFieldEventClosureType, PermissionArgs> =
  {
    return Event<TextFieldEventClosureType, PermissionArgs>(sender: self)
  }()
  
  public lazy var shouldReturn: Event<TextFieldEventClosureType, PermissionArgs> =
  {
    return Event<TextFieldEventClosureType, PermissionArgs>(sender: self)
  }()
}
```

Of course, the real thing has a lot more code but we are only interested in its events and how to handle them.

Now, we need to implement the EventSource protocol; and we can do this in an extension to the TextField class.

```swift
extension TextField : EventSource
{
  public func add(closures: [TextFieldEventClosureType])
  {
    closures.forEach
    {
      switch $0
      {
        case .shouldBeginEditing(_):
          shouldBeginEditing += $0
        case .didBeginEditing(_):
          didBeginEditing += $0
        case .shouldEndEditing(_):
          shouldEndEditing += $0
        case .didEndEditing(_):
          didEndEditing += $0
        case .shouldChangeCharacters(_):
          shouldChangeCharacters += $0
        case .shouldClear(_):
          shouldClear += $0
        case .shouldReturn(_):
          shouldReturn += $0
      }
    }
  }
  
  public func remove(closures: [TextFieldEventClosureType])
  {
    closures.forEach
    {
      switch $0
      {
        case .shouldBeginEditing(_):
          shouldBeginEditing -= $0
        case .didBeginEditing(_):
          didBeginEditing -= $0
        case .shouldEndEditing(_):
          shouldEndEditing -= $0
        case .didEndEditing(_):
          didEndEditing -= $0
        case .shouldChangeCharacters(_):
          shouldChangeCharacters -= $0
        case .shouldClear(_):
          shouldClear -= $0
        case .shouldReturn(_):
          shouldReturn -= $0
      }
    }
  }
}
```

Because the closures can be for any of the seven events, we need to use a switch statement to determine what each closure's enum type is and, then, add it to the approriate event in the same way that we did for the NotifyPropertyChanged protocol.

### A Delegate for the TextField's Events

UITextField's delegate property expects us to connect it to a complete object that implements the UITextFieldDelegate protocol. However, we could ask the question, why assign a whole object when you only need to handle one or two events?

Having now implemented the EventSource protocol in the TextField class, we no longer need to assign the whole delegate object, just a list of the appropriate closures for those events we want to handle.

So, next we need to define a delegate protocol, to be implemented on the delegate class, which will allow us to surface the list of closures that an EventSource needs to hook up to.

```swift
public protocol TextFieldEventDelegate
{
  var textFieldEventClosures: [TextFieldEventClosureType] { get }
}
```

There is not much to it; just a readonly property that we will have to implement on the delegate class.

```swift
extension TestClass : TextFieldEventDelegate
{
  public func textFieldShouldBeginEditing(sender: TextField, args: PermissionArgs)
  {
    …
  }
  
  public func textFieldDidBeginEditing(sender: TextField, args: EventArgs)
  {
    …
  }
  
  public func textFieldShouldEndEditing(sender: TextField, args: PermissionArgs)
  {
    …
  }
  
  public var textFieldEventClosures: [TextFieldEventClosureType]
  {
    return [.shouldBeginEditing(textFieldShouldBeginEditing),
            .didBeginEditing(textFieldDidBeginEditing),
            .shouldEndEditing(textFieldShouldEndEditing)]
  }
}
```

In this example, we are only interested in handling the `shouldBeginEditing`, `didBeginEditing` and `.shouldEndEditing` events, so we declare handling methods with the appropriate signatures and implement the TextFieldEventDelegate protocol in an extension to TestClass.

We implement the `textFieldEventClosures` property by returning an array that only contains a closure enum for each of the handling methods.

### The Test Code

At this point, it would be useful to add a method to the TextField class that calls one or more of the events and to do something like `print(…)`in the handler methods.

```swift
{
  func test()
  {
    let testObject = TestClass()
    
    let textField = TextField()
    
    textField.add(closures: testObject.textFieldEventClosures)
    
    textField.callTheEvents()
    
    textField.remove(closures: testObject.textFieldEventClosures)
  }
}
```

Of course, we could still use the `+=` operators on the individual events if we only needed to handle one or two of the events.

```swift
{
  func test()
  {
    let testObject = TestClass()
    
    let textField = TextField()
    
    textField.didBeginEditing += .didBeginEditing(testObject.textFieldDidBeginEditing)
    
    textField.callTheEvents()
    
    textField.didBeginEditing -= .didBeginEditing(testObject.textFieldDidBeginEditing)
  }
}
```

### Summary

After having designed a system that allowed us to assign methods or closures to individual events, this article went on to describe how we could link, not a whole delegate object to the subject object, but only a list of the methods or closures for the events in which we were interested in handling.

Now, if only Apple would adopt this 😎
