---
layout: post
title: Interactors - Part 2
---

### Introduction

So far, in this series, we have looked at the basics of Interactors and designed a couple of simple examples that take advantage of the NotifyPropertyChanged protocol to provide automatic updating of UILabels when an object's properties change value.

Now we move on to the, somewhat trickier, subject of getting an interactor to mediate in the other direction between an input **control**, such as UITextField, and the property of an object. Whereas the label interactors where relatively simple to implement, only needing to translate a value into text and update the view, this article will look at:

  1. reacting to user input in a control, such as a UITextField
  2. verifying keystrokes are valid
  3. verifying that the resulting text is "grammatically" correct
  4. translating the text into a meaningful value for the subject object

### The Basics

```swift
public protocol ControlInteractorProtocol : ViewInteractorProtocol
{
  func updateSubject()
  
  func resetSubject()
}
```

We start with a protocol that extends the `ViewInteractor` protocol to provide requirements notifying the subject of changes in response to user interaction.

I use the names ViewInteractorProtocol and ControlInteractorProtocol, because they align with the UIKit concept of a UIView, which is intended solely for displaying, and UIControl, which is intended to respond to user interaction.

### A Base Text Field Interactor

We start off by defining a base interactor class, which defines the view property from ViewInteractorProtocol as a UITextField.

```swift
@IBDesignable public class TextFieldInteractor : NSObjectPropertyInteractor, ControlInteractorProtocol
{
  @IBOutlet public var view: UITextField!
  {
    didSet
    {
      configureView()
    }
  }
  
  @IBInspectable public var shouldClear: Bool = true
  
  @IBInspectable public var shouldReturn: Bool = true
  
  override func didSet(subject: NSObject?)
  {
    super.didSet(subject: subject)
    
    updateView()
  }
  
  var keyboardType: UIKeyboardType
  {
    return view.keyboardType
  }
  
  func createValidCharacters() -> CharacterSet?
  {
    return nil
  }
  
  lazy var validCharacters: CharacterSet? =
  {
    return self.createValidCharacters()
  }()
  
  override func propertyDidChange(sender: Any, args: PropertyChangedEventArgs)
  {
    DispatchQueue.main.async
    {
      [unowned self] in
      
      if self.isActive
      {
        self.updateView()
      }
    }
  }
  
  public func updateSubject() { }
  
  public func resetSubject() { }
  
  public func configureView()
  {
    view.delegate = self
    
    view.keyboardType = keyboardType
    
    view.clearButtonMode = .whileEditing
  }
  
  public func updateView()
  {
    guard let subject = subject,
          let key = propertyName else
    {
      return
    }
    
    view.text = subject.value(forKey: key).description
  }
}
```

This class surfaces some of the UITextField's properties, such as: shouldClear, shouldReturn and keyboardType

We override the default empty configureView() method from ViewInteractorProtocol to configure the UITextField by:

  1. assigning the interactor to its delegate property
  2. setting its keyboardType of that of the interactor
  3. setting its clearButtonMode

The didSet(subject:) method, the propertyDidChange(sender:args:) method and the updateView() method are all implemented in much the same way as for the base LabelInteractor class, by displaying a string description of the property's value.

Additional stuff includes a lazy var to create a validCharacters CharacterSet property, which we will use for validating keystrokes in subclasses; here it simply returns the result of the base createValidCharacters() method that can be overridden in those subclasses to provide an appropriate character set for the property value type. In this base class we just return nil as, for a string value, we could say that any character is valid.

Both the updateSubject() and resetSubject() methods are implemented as empty since, at this stage, it isn't possible to know how they should work.

### Implementing the UITextField Delegate Protocol

Since we are going to need to react to various events from the UITextField, we create an extension to the TextFieldInteractor class to handle the events that are of interest to us.

```swift
extension TextFieldInteractor : UITextFieldDelegate
{
  public func textFieldDidBeginEditing(_ textField: UITextField)
  {
    if isActive
    {
      deactivate()
    }
  }
  
  public func textFieldDidEndEditing(_ textField: UITextField)
  {
    updateView()
    
    if !isActive
    {
      activate()
    }
  }
  
  public func textFieldShouldClear(_ textField: UITextField) -> Bool
  {
    resetSubject()
    
    return shouldClear
  }
  
  public func textFieldShouldReturn(_ textField: UITextField) -> Bool
  {
    updateSubject()
    
    if shouldReturn
    {
      textField.resignFirstResponder()
    }
    
    return shouldReturn
  }
}
```

The textFieldDidBeginEditing(\_:) method deactivates the interactor to avoid subject updates interrupting user input in the control. This is a subject that could be discussed further, with questions like: should we allow interruption or should we signal a change? Such discussion would complicate this article and is left to the reader to have.

The textFieldDidEndEditing(\_:) method updates the view from the subject's property and reactivates the interactor.

The textFieldShouldClear(\_:) method resets the subject's property to its default value and returns the shouldClear property from the interactor.

Finally, the textFieldShouldReturn(\_:) method does a final update the subject's property value and resigns the control as firstResponder, thus hiding the keyboard, the cursor and the clear button from the text field.

There may be other events that you would want to handle here, or even provide target/action methods to capture notifications from the text field; those are left to the reader's choice but the principle of how so to do should be evident.

### Dealing with Optional Property Values

Because we want to be able to allow the use of optional properties in the subject object, I wanted to make it easy to provide a string to represent a optional property value, whether it contained a value or nil.

extension Optional : CustomStringConvertible
{
  public var description: String
  {
    if case .some(let value) = self
    {
      return String(describing: value)
    }
    
    return ""
  }
}

This extension to the Optional enum mixes in the CustomStringConvertible protocol to provide a description property to any optional var. It simply checks whether the enum is `.some` and returns a string desribing the value or, if not, it returns an empty string.

There are other ways of dealing with this situation but, for the sake of avoiding too much distraction in this article, I chose this.

### Setting the Value of an Optional Property

To truly cope with resetting optional properties, we really need to know whether the property's type is optional or not in order to decide whether to set it to nil or to the default value for the property's type.

Not wanting to use Objective-C's runtime functionality, with all its limitations and complexity, I decided to create a small dictionary that has the names of a subject's properties for keys and the DisplayStyles of the properties, as provided by Swift's Mirror struct, for values; if a property is optional, the DisplayStyle is set to "optional".

I added a lazy var to the NSObjectPropertyInteractor class, in order that it can be used by any interactor that derives from that class:

```swift
@IBDesignable public class NSObjectPropertyInteractor : Interactor, PropertyInteractorProtocol
{
  …
  
  lazy var propertyDisplayStyles: [String : Mirror.DisplayStyle]? =
  {
    guard let subject = self.subject else
    {
      return nil
    }
    
    var propertyDisplayStyles: [String : Mirror.DisplayStyle] = [:]
    
    let mirror = Mirror(reflecting: subject)
    
    mirror.children.forEach
    {
      child in
      
      guard let name = child.label else
      {
        return
      }
      
      let childMirror = Mirror(reflecting: child.value)
      
      propertyDisplayStyles[name] = childMirror.displayStyle
    }
    
    return propertyDisplayStyles
  }()
  
  …
}
```

Unfortunately, Swift's Mirror doesn't yet allow us to "reflect" on a type to get its metadata, so we end up having to use our subject instance, just to get the type information. Let's hope Apple can improve on this this soon.

By creating this dictionary as a lazy var, we get the double advantage of not creating the dictionary until it is first needed and only creating it once, effectively caching it.

### Default Type Values

As well as dealing with the displaying of optional property values, we are also going to need to provide a default value for any property when we implement the resetSubject() method.

```swift
protocol DefaultValue
{
  static var defaultValue: Self { get }
}

extension ExpressibleByIntegerLiteral
{
  static var defaultValue: IntegerLiteralType
  {
    return 0 as! Self.IntegerLiteralType
  }
}
```

The DefaultValue protocol is not strictly necessary for types like Int, Double, etc; all we need to do is to extend the ExpressibleByIntegerLiteral protocol to add the defaultValue static var. I tried extending ExpressibleByFloatLiteral as well, but this seemed to confuse the compiler and it appears that float default values can be expressed in terms of ExpressibleByIntegerLiteral.

### An Integer Control Interactor

Because classes that handle control interaction can be quite complex, I will split the code for the IntegerControlInteractor into sections, commenting on each section in turn.

```swift
@IBDesignable public class IntegerTextFieldInteractor : TextFieldInteractor
{
  public override func configureView()
  {
    super.configureView()
    
    view.spellCheckingType = .no
    
    view.autocorrectionType = .no
    
    view.textAlignment = .right
  }
  
  public override func updateView()
  {
    guard let subject = subject,
          let key = propertyName else
    {
      return
    }
    
    view.text = subject.value(forKey: key).description
  }
  
  …
```

We start by adding further configuration to the text field, by disabling spell-checking and auto-correction and setting the text alignemt to the right.

For the updateView() method, because we are dealing only with integers and the characters in the text will already have been validated, possibly the simplest way to provide a textual representation of the, possibly optional, property value was to use the extension to Optional enum described above. 

```swift
  …
  
  public override func updateSubject()
  {
    guard let subject = subject,
          let key = propertyName else
    {
      return
    }
    
    if let text = view.text,
       !text.isEmpty
    {
      subject.setValue(Int(text), forKey: key)
    }
    else
    {
      resetSubject()
    }
  }

  public override func resetSubject()
  {
    guard let subject = subject,
          let key = propertyName,
          let displayStyle = propertyDisplayStyles?[key] else
    {
      return
    }
    
    if displayStyle == .optional
    {
      subject.setValue(nil, forKey: key)
    }
    else
    {
      subject.setValue(Int.defaultValue, forKey: key)
    }
  }
  
  …
```

In the updateSubject() method, we test to see whether there is any text in the text field and, if so, we create an Int from it and pass that to the subject.

If there is no text, then we can use the same code that we would for resetting the property, which takes into account if no text during editing translates to nil or a default value being shown when the text field is not being edited, so we simply call resetSubject().

The resetSubject method uses the propertyDisplayStyles dictionary to determine the displayStyle for the interactor's property and, if it is .optional, then set the property to nil, otherwise, set it to the defaultValue for the type.

```swift
  …
  
  public override var keyboardType: UIKeyboardType
  {
    return .numberPad
  }
  
  override func createValidCharacters() -> CharacterSet?
  {
    return CharacterSet.decimalDigits
  }
  
  …
```

We use the .numberPad keyboard for integer entry, even though, on iPads, this still shows the full keyboard, which is why we are going to have to validate every keystroke.

For integer text entry, the only characters we need here are included in the CharacterSet.decimalDigits set; this could be different if we wanted to deal with negative values as well.

##### Validating Keystrokes

Now comes the most complex method of this type of validator; although the integer validation code is slightly less complex than that for decimal numbers.

I have added comments inline to try to explain what is going on.

```swift
  public func textField(_ textField: UITextField,
                        shouldChangeCharactersIn range: NSRange,
                        replacementString string: String) -> Bool
  {
    guard let subject = subject,
          let key = propertyName else
    {
      return false
    }
      
    // guarantee we have, at least, an empty string to work with
    var text = textField.text ?? ""
    
    // if the string parameter is empty, this indicates the backspace key has been pressed
    if string.isEmpty
    {
      // remove last character from text
      text = text.substring(to: text.index(before: text.endIndex))
      
      if !text.isEmpty
      {
        // convert text to Int and assign it to property
        subject.setValue(Int(text), forKey: key)
      }
      else
      {
        // get DisplayStyle for this interactor's property
        guard let displayStyle = propertyDisplayStyles?[key] else
        {
          return false
        }
        
        // no text left - set property to either nil or default value
        if displayStyle == .optional
        {
          subject.setValue(nil, forKey: key)
        }
        else
        {
          subject.setValue(Int.defaultValue, forKey: key)
        }
      }
      
      return true
    }
    
    // from here on, we are dealing with added characters in the string parameter
    
    // validate the string parameter
    guard let validCharacters = validCharacters,
          string.rangeOfCharacter(from: validCharacters) != nil else
    {
      return false
    }
    
    // if the text only contains a default value of 0, delete it
    if text == "0"
    {
      textField.deleteBackward()
    }
    
    // insert the contents of the string parameter at the insertion point
    // this should not fail here but you never know
    guard let value = Int(text.stringByInserting(string, at: range)) else
    {
      return false
    }
    
    // update the property value
    subject.setValue(value, forKey: key)
    
    return true
  }
```

### Summary

No matter how many times I go over the validation code in this and the decimal interactor, I still get the sneaky feeling I might just have missed a corner case.

If you should find it, please contact me to let me know.

Otherwise, it's time to move on to the next article, where the validation code gets even scarier.
