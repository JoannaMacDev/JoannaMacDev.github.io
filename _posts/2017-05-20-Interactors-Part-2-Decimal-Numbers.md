---
layout: post
title: Interactors - Part 2
---

### Introduction

This is the final article in this series on interactors for the present. It contains, by far, the most complex validation code, which I invite you to test to the extreme, looking tof that one edge case that I may have missed.

### A Decimal Control Interactor

Now we come to the most complex interactor so far; one that validates text entry for decimal value properties.

Once again, I will split the code for the class into sections, with comments both in the code and between the sections.

Let's start this section by declaring a small helper extension the Array struct, that will allow us use meaningful enum names to access the two parts of a string represented a decimal number.

Here is the enum:

```swift
private enum DecimalPart : Int
{
  case integer
  case fraction
}
```

And here is the extension to Array

```swift
private extension Array
{
  subscript(index: DecimalPart) -> Element
  {
    return self[index.rawValue]
  }
}
```

We can also extend the CharacterSet struct to add a static let to give us the decimal separator, as defined by the current locale.

```swift
extension CharacterSet
{
  static let decimalSeparator : CharacterSet =
  {
    let decimalSeparator = Locale.autoupdatingCurrent.decimalSeparator ?? "."
    
    let decimalSeparatorCharacterSet = CharacterSet(charactersIn: decimalSeparator)
    
    return decimalSeparatorCharacterSet
  }()
}
```

We're going to start the actual class definition by deriving from the IntegerTextFieldInteractor.

```swift
@IBDesignable public class DecimalTextFieldInteractor : IntegerTextFieldInteractor
{
  @IBInspectable public var maximumIntegerDigits: Int = 8
  
  @IBInspectable public var minimumFractionDigits: Int = 1
  
  @IBInspectable public var maximumFractionDigits: Int = 3
  
  private lazy var formatter: NumberFormatter =
  {
    let formatter = NumberFormatter()
    
    formatter.locale = Locale.autoupdatingCurrent
    
    formatter.minimumIntegerDigits = 1
    
    formatter.maximumIntegerDigits = self.maximumIntegerDigits
    
    formatter.minimumFractionDigits = self.minimumFractionDigits
    
    formatter.maximumFractionDigits = self.maximumFractionDigits
    
    return formatter
  }()
  
  public override var keyboardType: UIKeyboardType
  {
    return .decimalPad
  }
  
  …
```

We have added three @IBInspectable properties to enable a developer to set up the preferred decimal format in the IB designer.

Then we have a NumberFormatter, held in a lazy var, which uses the three properties as part of its configuration.

The keyboard type is set to .decimalPad, even though this will still leave us with a full keyboard on an iPad; hence the need for even more keystroke validation.

```swift
  …
  
  public override func updateView()
  {
    guard let subject = subject,
          let key = propertyName,
          let value = subject.value(forKey: key) as? Double else
    {
      return
    }
    
    view.text = formatter.string(from: NSNumber(value: value))
  }
  
  public override func updateSubject()
  {
    guard let subject = subject,
          let key = propertyName else
    {
      return
    }
    
    if let text = view.text,
       !text.isEmpty,
       let number = formatter.number(from: key)
    {
      subject.setValue(number.doubleValue, forKey: key)
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
      subject.setValue(Double.defaultValue, forKey: key)
    }
  }
  
  override func createValidCharacters() -> CharacterSet?
  {
    guard var validCharacters = super.createValidCharacters() else
    {
      return super.createValidCharacters()
    }
    
    validCharacters.formUnion(.decimalSeparator)
    
    return validCharacters
  }
  
  …
```

The updateView(), updateSubject() and resetSubject() methods are almost identical to those for the integer version with only the value types being different and the use of the formatter in the updateView() method.

The createValidCharacters() method override takes the result of the integer version and just adds the decimal separator.

##### Validating Keystrokes

So, now we come to the complicated (could we factor it out?) key validation method:

```swift
  …
  
  public override func textField(_ textField: UITextField,
                                 shouldChangeCharactersIn range: NSRange,
                                 replacementString string: String) -> Bool
  {
    // guarantee we have, at least, an empty string
    var text = textField.text ?? ""
    
    // check for the existence of a valid decimal separator for the current locale
    guard let decimalSeparator = Locale.autoupdatingCurrent.decimalSeparator else
    {
      return false
    }
    
    // if the string parameter is empty, this indicates the backspace key has been pressed
    if string.isEmpty
    {
      // during deletion, "0" on its own is invalid if it is not followed by a decimal
      // separator therefore, if the next character to delete is the decimal separator
      // we need to delete the 0 as well
      if text.hasSuffix(decimalSeparator),
         text.hasPrefix("0")
      {
        // manually delete decimal separator
        textField.deleteBackward()
        
        // this will then delete the "0"
        return true
      }
      
      // update text to reflect deleted digit
      text = text.substring(to: text.index(before: text.endIndex))
      
      guard let subject = subject,
            let key = propertyName else
      {
        return false
      }
      
      if !text.isEmpty
      {
        if let number = formatter.number(from: text)
        {
          subject.setValue(number.doubleValue, forKey: key)
        }
      }
      else
      {
        guard let displayStyle = propertyDisplayStyles?[key] else
        {
          return false
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
      
      return true
    }
    else
    {
      // we are adding new characters
      
      // test for invalid characters
      guard let validCharacters = validCharacters,
            string.rangeOfCharacter(from: validCharacters) != nil else
      {
        return false
      }
      
      // disallow multiple decimal separators
      if (string.rangeOfCharacter(from: .decimalSeparator) != nil) && text.rangeOfCharacter(from: .decimalSeparator) != nil
      {
        return false
      }
      
      // delete extraneous leading 0 before anything other than the decimal separator
      if text == "0",
         string != decimalSeparator
      {
        textField.deleteBackward()
      }
      
      // insert new character(s) to contents of text field at current insertion point
      // indicatd by the range parameter passed to this method
      text = text.stringByInserting(string, at: range)
      
      // separate out text into integer part and fraction part
      let textComponents = text.components(separatedBy: decimalSeparator)
      
      // limit number of integer digits unless next character is the decimal separator
      if textComponents[.integer].characters.count > maximumIntegerDigits,
         string != decimalSeparator
      {
        return false // we could also provide feedback here
      }
      
      if textComponents.count > 1,
         textComponents[.fraction].characters.count > maximumFractionDigits
      {
        return false // we could also provide feedback here
      }
      
      // if the first character of text is the decimal separator, set the text field's text to "0"
      // returning true will add the decimal separator and the fraction part if there is one
      if text.hasPrefix(decimalSeparator)
      {
        textField.insertText("0")
        
        if string == decimalSeparator
        {
          text = "0"
        }
      }
    }
    
    // in theory, this should not fail, but we don't all live in Theory
    guard let value = formatter.number(from: text.isEmpty ? "0" : text) else
    {
      return false
    }
    
    subject.setValue(value, forKey: key)
    
    return true
  }
}
```

### Summary

I hope you have enjoyed this series on interactors. Of course, there will be all sorts of other views and controls that you may wish to build an interactor for; I may return to the topic if I find a particularly interesting one to share with you.

To recap, the main aim of interactors is to keep key validation code out of the ViewController and to avoid having to write the same or similar code time and time again. In addition, once you have a library of these interactors built up, it is so simple to drop one on a view controller and hook it up.

Now, what shall we look at next?
