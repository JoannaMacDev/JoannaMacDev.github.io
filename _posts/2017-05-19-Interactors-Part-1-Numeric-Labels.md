---
layout: post
title: Interactors - Part 1 - Numeric Labels
---

### Introduction

Just a small article to separate out the handling of numeric values in interactors derived from LabelInteractor.

### An Integer Label Interactor

```swift
@IBDesignable public class IntegerLabelInteractor : LabelInteractor
{
  public override func updateView()
  {
    guard let subject = subject,
          let key = propertyName else
    {
      return
    }
    
    let value = subject.value(forKey: key)
    
    if case .some(let nonNilValue) = value
    {
      view.text = String(nonNilValue as! Int)
    }
    else
    {
      view.text = "«nil»"
    }
  }
}
```

All we need to do with this interactor is to override the updateView() method to correctly handle an integer value; or, to be more precise, an optional integer value.

It is important to realise that, in some classes, there is no default value for some properties, apart from nil.

Because we are using NSObject's value(forKey:) method to retrieve the value, what we are receiving is actually an Optional<Any>. Therefore, we are using an `if case .some(let …)` to verify that there is a value end, if there is, to extract it. Then we simply need to force the cast from Any to Int and create a string from it.

If the value is nil, then we can either display an empty string or, as I have suggested here, a sring that tells the user that the value is "nil". What we display will depend on what the user would be most likely to expect.

### A Decimal Label Interactor

```swift
@IBDesignable public class DecimalLabelInteractor : LabelInteractor
{
  @IBInspectable public var maximumIntegerDigits: Int = 5
  
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
  
  public override func updateView()
  {
    guard let subject = subject,
          let key = propertyName else
    {
      return
    }
    
    let value = subject.value(forKey: key)
    
    if case .some(let nonNilValue) = value
    {
      view.text = formatter.string(from: NSNumber(value: nonNilValue as! Double))
    }
    else
    {
      view.text = "«nil»"
    }
  }
}
```

In the case of the handling of decimal numbers, we need to take into consideration a few more factors.

Here, we have three @IBInspectable properties that allow us to use the IB designer to specify the formatting of a string representing a decimal number, in terms of how many digits will appear before and after the decimal separator.

Of course, we will need a NumberFormatter to make the conversion from number to string and, here, we create it as a lazy var. Note that the locale is specified as autoupdatingCurrent to ensure that the decimal and thousands spearators are correct for the users device region.

Again, we need to handle the possibility of nil values, in the same way that we did for the integer interactor.

### Summary

All in all, displaying of numeric values is a relatively simple matter and it could still be argued that interactors could be excessive, apart fromt eh fact that they do allow us to encapsulate the code that reacts to property value changes and the formatting of text in one, easily reusable, object.

Coming next, the ever so slightly more complex matter of validating keystrokes, copying, pasting, etc of numeric values as they are entered into a UITextField.
