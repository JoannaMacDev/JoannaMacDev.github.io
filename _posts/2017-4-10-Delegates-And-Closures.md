---
layout: post
title: Delegates vs Closures in Swift
---

### The Observer Pattern

Basically speaking, the Observer Pattern describes a coding pattern that facilitates the notification of a change in one object to one or more **observing** objects.

Apple's AppKit and UIKit frameworks are both built around the Model View Controller (MVC) design pattern but, although we are used to working with delegates to allow the Controller part of MVC to react to the View part of MVC, there is comparatively little written about getting the Controller to react to the Model part of MVC.

When it comes to reacting to the UI in Objective-C, Apple gives us three principal mechanisms:

1. Delegates
2. Target-Action
3. NotificationCenter

Swift adds a fourth - Closures.

##### Delegates

For those who use Objective-C, the concept of delegates will be familiar. Probably the first delegates most of us would have encountered would have been UITableViewDataSource and UITableViewDelegate; both of which serve as means to customising the behaviour of a UITableView.

Most of the time, our idea of delegates is something that is used to give us an opportunity to "participate" in the life of a UI component, either allowing, or not, something to happen in the UI component, providing information required by the UI component or reacting to a change in state in the UI component.

Delegate code is performed synchronously.


##### Target-Action

Whereas a delegate usually contains more than one delegating method (action), it can only point to one delegate object (target), Target-Action allows us add multiple targets and multiple actions to a UI component. You are implementing Target-Action when you create @IBAction methods to react to buttons, menu items, touch events, etc.

But, once again, Target-Action is primarily aimed at getting a controller to react to an activity in a UI component; indeed it is based around UIControl or NSControl and the @IBAction marker is for the benefit of the designers in Interface Builder.

Target-Action code is performed synchronously.


##### NotificationCenter

Similar to Target-Action, but not just for UI controls, NotificationCenter is a **message-based** system that allows us to notify multiple **observers**.

Instead of being primarily aimed at responding to UI activity, NotificationCenter is available for notifications to be sent from an object of any type to observing objects of any type, provided the observing objects are registered with the NotificationCenter to receive notifications; this makes it useful for notifying a Controller of changes in a Model.

NotificationCenter code is performed synchronously, unless a NotificationQueue is used.

### Delegates - pros & cons

The Apple idea of delegates has its roots in Objective-C and is very simplistic - e.g.

*Objective-C*
```objectivec
@protocol UITextFieldDelegate <NSObject>

@optional

- (BOOL)textFieldShouldBeginEditing:(UITextField *)textField;
- (void)textFieldDidBeginEditing:(UITextField *)textField;
- (BOOL)textFieldShouldEndEditing:(UITextField *)textField;
- (void)textFieldDidEndEditing:(UITextField *)textField;
- (void)textFieldDidEndEditing:(UITextField *)textField reason:(UITextFieldDidEndEditingReason)reason
- (BOOL)textField:(UITextField *)textField shouldChangeCharactersInRange:(NSRange)range replacementString:(NSString *)string;
- (BOOL)textFieldShouldClear:(UITextField *)textField;
- (BOOL)textFieldShouldReturn:(UITextField *)textField;

@end
```
*Swift*
```swift
public protocol UITextFieldDelegate : NSObjectProtocol
{
  optional public func textFieldShouldBeginEditing(_ textField: UITextField) -> Bool
  optional public func textFieldDidBeginEditing(_ textField: UITextField)
  optional public func textFieldShouldEndEditing(_ textField: UITextField) -> Bool
  optional public func textFieldDidEndEditing(_ textField: UITextField)
  optional public func textFieldDidEndEditing(_ textField: UITextField, reason: UITextFieldDidEndEditingReason)
  optional public func textField(_ textField: UITextField, shouldChangeCharactersIn range: NSRange, replacementString string: String) -> Bool
  optional public func textFieldShouldClear(_ textField: UITextField) -> Bool
  optional public func textFieldShouldReturn(_ textField: UITextField) -> Bool
}
```

Every method contains a parameter that represents the delegating object and, optionally, parameters that provide additional information about the action that is being observed by the delegate.

Some methods expect a value to be returned (in this case mainly a boolean indicating whether the action should occur or not. the rest are informational and tell us when something has happened.

##### Pros

Every method passes the **sender** as its first parameter, allowing us to know its type and contents

##### Cons

This protocol is explicitly designed, only for use with UITextField, as are most such protocols - only designed as a delegate for one specific type

### Roll-your-own Delegates

*Objective-C*
```objectivec
@protocol NameChangeDelegate

- (void)nameDidChange:(id)sender name:(NSString *)name;

@end
```
*Swift*
```swift
public protocol NameChangeDelegate
{
  func nameDidChange(sender: Any, name: String?)
}

public class SimpleSubject
{
  public var nameChangeDelegate: NameChangeDelegate?
  
  public var name: String?
  {
    didSet
    {
      nameChangeDelegate?.nameDidChange(sender: self, name: name)
    }
  }
}

public class SimpleDelegate : NameChangeDelegate
{
  public func nameDidChange(sender: Any, name: String?)
  {
    if let sender = sender as? SimpleSubject
    {
      print("\(sender) : \(name)")
    }
  }
}
```

At a quick glance, all seems well but, on further inspection, we find that the sender parameter is **id** (Objective-C) or **Any** (Swift); so, when we implement the method in the delegate object, we have no indication of the real type of the sender and no easy way of inspecting its contents; at least without a massive series of `if let sender = sender as? …` statements, a switch statement, or similar.

##### So, how about using generics?

Objective-C doesn't yet have first-class generics, so there is not much more we can do, apart from trying to determine the type of the sender in the implementing delegate method

However, since this article is primarily about Swift coding, and Swift does give us generics, can't we simply define an associated type in the protocol and use that for the sender parameter?

Well, not unless you want to enter into the world of pain that comes with Swift's implementation of "generic" protocols. Here's the problem in a nutshell.

As soon as we do this…

```swift
public protocol NameChangeDelegate
{
  associatedtype SenderType
  
  func nameDidChange(sender: SenderType, name: String?)
}
```

We end up with this…

```swift
public class SimpleSubject
{
  public var nameChangeDelegate: NameChangeDelegate? // error : Protocol 'NameChangeDelegate' can only be used as a generic constraint because it has Self or associated type requirements
  
  public var name: String?
  {
    didSet
    {
      nameChangeDelegate?.nameDidChange(sender: self, name: name) // error : Member 'nameDidChange' cannot be used on value of protocol type 'NameChangeDelegate'; use a generic constraint instead
    }
  }
}

public class SimpleDelegate : NameChangeDelegate // error : Type 'SimpleDelegate' does not conform to protocol 'NameChangeDelegate'
                                                 // error : Protocol requires nested type 'SenderType'; do you want to add it?
{
  public func nameDidChange(sender: SenderType, name: String?) // error : Use of undeclared type 'SenderType'
  {
    print("\(sender) : \(name)")
  }
}
```

In a word, Aaaarrrgghhh!!!. What this essentially boils down to is that we have to know the real type of the sender wherever it is used, even though we were hoping, as a generic type, it could be inferred. So, we end up with all sorts of convoluted code (known as **type erasure** to get the SimpleSubject class to compile; something like this…

```swift
public class NameChangeDelegateWrapper<delegateT : NameChangeDelegate>
{
  private let sender: delegateT.SenderType
  
  private let wrappedValue: delegateT
  
  init(sender: delegateT.SenderType, wrappedValue: delegateT)
  {
    self.sender = sender
    
    self.wrappedValue = wrappedValue
  }
  
  func invoke(name: String?)
  {
    wrappedValue.nameDidChange(sender: sender, name: name)
  }
}

public class SimpleSubject<delegateT : NameChangeDelegate>
{
  public var nameChangeDelegate: NameChangeDelegateWrapper<delegateT>?
  
  public var name: String?
  {
    didSet
    {
      nameChangeDelegate?.invoke(name: name)
    }
  }
}
```

That might solve the SimpleSubject side of things, but it still leaves us with an unholy mess in the SimpleDelegate class, where we end up, essentially, with a similar situation to when the sender was passed as an Any. Trust me, there isn't enough room in this article to try and work it out.

### Building a Better Mousetrap

So, how would it be if we left behind the concept of delegates for notifying observers of changes to an object and changed, instead, to using the "new kid on the block" of Swift closures? And what if we combined the use of closures with the Target-Action ability to add more than one closure (action) on more than one observer (target)?

And, whilst we are doing all that, why not change the observing closure's (non-sender) parameters to a single **parameter object**?

##### The NotifyPropertyChanged Protocol

I'm borrowing the INotifyPropertyChanged interface (protocol) from C#, as an example of how we can do all this wonderful stuff. We start by declaring the protocol…

```swift
public protocol NotifyPropertyChangedProtocol
{
  var propertyChanged: Event<Self, PropertyChangedEventArgs> { get set }
}
```

We pass Self as the first generic parameter to the Event<senderT, argsT> type because the sender type will always be the implementing type for this protocol.

Now we need to fill in the types used within the protocol; let's start with a base parameterless class that can be derived from, for use as the **args** object that will get passed to the closures…

```swift
public class EventArgs
{
  public static let empty: EventArgs = EventArgs()
}
```

Now we can create our PropertyChangedEventArgs class…

```swift
public class PropertyChangedEventArgs : EventArgs
{
  public static let allProperties = PropertyChangedEventArgs(nil)
  
  public let propertyName: String?
  
  public init(_ propertyName: String?)
  {
    self.propertyName = propertyName
  }
}
```

Instances of this class will hold the name of the property within the subject that has changed. The `allProperties` static let is to allow us to use an instance of this class, with no name, to indicate that more than one property has changed; thus indicating that we need to examine the whole object to determine what has changed. We could also change the initialiser to take an array of Strings instead, an empty array indicating all properties had changed.

The next thing we are going to need is a type of closure that will handle the change:

```swift
public typealias PropertyChangedEventClosure<senderT> = (senderT, PropertyChangedEventArgs) -> ()
```

We don't actually need to use this typealias in this example, but it can be useful to see what the parameter types should be when we come to add a closure to handle the event.

#### The Main Event

Next, we need to define the **event** type that is part of the NotifyPropertyChangedProtocol; this is a generic type that will work with any "args" class derived from EventArgs; it will allow us to add multiple closures and, when invoked, will call all of those closures in turn.

We use an EventClosure<senderT, argsT : EventArgs> typealias, which is totally generic, for the closure type, but equally supports the passing of the, more specialised, PropertyChangedEventClosure<senderT> closure, specific to our example.

```swift
public typealias EventClosure<senderT, argsT : EventArgs> = (senderT, argsT) -> ()
```

```swift
// MARK: operator overloads

public func +=<senderT, argsT>(_ event: inout Event<senderT, argsT>, handler: @escaping EventClosure<senderT, argsT>)
{
  event.add(handler)
}

public func -=<senderT, argsT>(_ event: inout Event<senderT, argsT>, handler: @escaping EventClosure<senderT, argsT>)
{
  event.remove(handler)
}

public class Event<senderT, argsT : EventArgs>
{
  // MARK: private properties
  
  private lazy var eventHandlers: [EventClosure<senderT, argsT>] =
  {
    return [EventClosure<senderT, argsT>]()
  }()
  
  private var sender: senderT
  
  // MARK: - fileprivate methods, accessible from the operator overloads in this file
  
  fileprivate func add(_ handler: @escaping EventClosure<senderT, argsT>)
  {
    eventHandlers.append(handler)
  }
  
  fileprivate func remove(_ handler: @escaping EventClosure<senderT, argsT>)
  {
    // required to access index(of:_)
    guard let eventHandlersAsNSArray = eventHandlers as NSArray? else
    {
      return
    }
    
    let index = eventHandlersAsNSArray.index(of: handler)
    
    guard index != NSNotFound else
    {
      return
    }
    
    let _ = eventHandlers.remove(at: index)
  }
  
  // MARK: - contructors
  
  public init(sender: senderT)
  {
    self.sender = sender;
  }
  
  // MARK: - public methods
  
  public func invoke(with args: inout argsT)
  {
    for eventHandler in eventHandlers
    {
      eventHandler(sender, args)
    }
  }
}
```

We pass a reference to the the containing "subject" to the initialiser, so that we can hold onto it to pass to the closures when they are called.

The only peculiar part of this class is that, to find which closure to remove, we have to bridge the array of closures to NSArray, in order to use index(of:_) method.

### Putting the Event to Work

All that is left now is to add the event mechanism to a test class…

```swift
public class TestSubject : NotifyPropertyChangedProtocol
{
  lazy var propertyChanged : Event<TestSubject, PropertyChangedEventArgs> =
  {
    return Event<TestSubject, PropertyChangedEventArgs>(sender: self)
  }()
  
  public var name: String = ""
  {
    didSet
    {
      var args = PropertyChangedEventArgs("name")
      
      propertyChanged.invoke(with: &args)
    }
  }
}
```

The event is best held in a lazy var; it would not be possible to pass self to the event's initialiser, inside the subject's initialiser, due to self not having being fully initialised because the event has not yet been initialised (otherwise known as a Catch 22 situation)

Finally, for the subject class, we add an example property and use its didSet block to call the propertyChanged event.

Now we can create a test delegate class that contains a closure that matches the signature of the `PropertyChangedEventClosure<senderT>` typealias that we declared earlier.

```swift
public class TestDelegate
{
  func handlePropertyChanged(sender: TestSubject, args: PropertyChangedEventArgs)
  {
    print("\"\(sender.name)\" has been assigned to \(args.propertyName ?? "no property name")")
  }
}
```

Notice that we do not have to implement a protocol on the **delegate** class, only on the **subject**

Finally, we can write some test code…

```swift
{
  let testDelegate = TestDelegate()
  
  let testSubject = TestSubject()
  
  testSubject.propertyChanged += testDelegate.handlePropertyChanged
  
  testSubject.name = "Swift"
}
```

## Conclusion

And there you have it. a system that allows you to create any concrete class and add the ability to observe changes to instances of that type. The TestDelegate could also be declared as a struct and, with a few subtle tweaks to the TestSubject code, it too could be implemented as structs.

Here we are only concerned with the NotifyPropertyChangedProtocol but, the principle can be used to call closures for any reason. It is perfectly feasible to add multiple events to a type and add multiple closures from multiple delegate types.