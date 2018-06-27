---
layout: post
title: Swift Optional Protocol Methods
---

### Introduction

According to the Swift documentation, it is not possible to have optional methods in a protocol without declaring both the protocol and the methods as `objc`. This article presents one possible and relatively simple solution, or should that be workaround?

### Theoretical Protocol

```swift
protocol CommandExecution
{
  func doExecute(Bool) -> Bool
  
  optional func doExecuteFailure()
  
  optional func doExecuteSuccess()
}
```

### Declaring a Protocol with Optional Methods

The workaround is not necessarily pretty but it provides a means of declaring a method that is literally optional, since it uses an optional readonly var that returns either a closure or nil.

```swift
protocol ExecutableCommand
{
  // required
  func doExecute() -> Bool
}

extension ExecutableCommand
{
  // optional
  var doExecuteFailure: (() -> ())?
  {
    return nil
  }
  
  // optional
  var doExecuteSuccess: (() -> ())?
  {
    return nil
  }
}
```

The syntax for the optional methods may seem a little verbose but it actually comes fairly close to the C++ way of declaring a pure virtual method. Thus the equivalent of our protocol would be an abstract C++ class something like this:


```
class ExecutableCommand
{
  virtual bool doExecute() = 0
  
  virtual void doExecuteFailure() = 0
  
  virtual void doExecuteSuccess() = 0
}

void ExecutableCommand::doExecuteFailure() { }

void ExecutableCommand::doExecuteSuccess() { }
```

### Implementing the Protocol

First, for the purpose of this example, let's add the ability to randomise a boolean:

```swift
extension Bool
{
  static var random: Bool
  {
    return arc4random_uniform(2) == 0
  }
}
```

Now we can write a simple example struct that implements the `ExecutableCommand` protocol

```
struct Command : ExecutableCommand
{
  func execute()
  {
    if doExecute()
    {
      doExecuteSuccess?()
    }
    else
    {
      doExecuteFailure?()
    }
  }
  
  func doExecute() -> Bool
  {
    return Bool.random
  }
}
```

Here, we call the optional methods of the protocol in much the same way that we would expect to call any optional closure, by adding a `?` between the closure name and the parentheses.

Now we need some test code:

```
{
  let command = Command()
  
  for _ in 0..<10
  {
    command.execute()
  }
}
```

This runs fine, demonstrating that calls to the nil optional methods are correctly ignored but, unless we step through the code in the debugger, running this code doesn't really tell us anything about whether executing the Command succeeded or failed. So let's implement the optional methods to write something to the console:

```
struct Command : ExecutableCommand
{
  func execute()
  {
    if doExecute()
    {
      doExecuteSuccess?()
    }
    else
    {
      doExecuteFailure?()
    }
  }
  
  func doExecute() -> Bool
  {
    return Bool.random
  }
  
  var doExecuteFailure: (() -> ())?
  {
    return {
             print("Failure")
           }
  }
  
  var doExecuteSuccess: (() -> ())?
  {
    return {
             print("Success")
           }
  }
}

```

Run the test code again and you should see a list of randomly generated Failure or Success in the console.

### Summary

Personally, I'm quite pleased with the syntax for using these optional methods, in some ways quite similar to C++, just different in how the code is laid out.

I'd be interested to know what you think. You can leave a message on Twitter - @JoannaMacDev
