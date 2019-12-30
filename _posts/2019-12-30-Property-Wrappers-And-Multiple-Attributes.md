---
layout: post
title: Property Wrappers and Multiple Attributes
---

### Introduction

Ever since leaving C# behind, I have missed the concept of Attributes for types, methods and properties. Slowly but surely, Swift has started to add very limited functionality in this area, especially from a custom attributes point of view; but even then, we are limited to Property Wrapper Attributes and then only one attribute per property. This article seeks to explain a way to provide multiple attributes for one property, using existing Swift mechanisms.  

### It All Starts With a Protocol

Here's a base empty protocol, which all attributes have to implement. Think of it as a base type which we can use to hold a heterogeneous collection of differing attributes.

```
protocol Attribute
{
  
}
```

Now, let's create a simple attribute, which will allow us to declare whether a property is to be persisted or not:

```
enum PersistencyAttribute : Attribute
{
  case persistent
  case transient
}
```

Now for something more complex, which will allow us to attach a validation closure to a property:

```
struct ValidationAttribute<valueT> : Attribute
{
  let closure: ((valueT?) -> (Bool))
}
```

Now for the property wrapper that will allow us to make use of these multiple attributes (or others) on a single property:

We start with the simplest of property wrappers that simply wraps a property:

```
@propertyWrapper
struct Attributes<typeT>
{
  private var value: typeT?
  
  ...
  
  var wrappedValue: typeT?
  {
    get
    {
      return value
    }
    set
    {
      value = newValue
    }
  }
  
  ...
}
```

Now we need to add a way of holding on to the attributes that we want to attach to a given property:

```
@propertyWrapper
struct Attributes<valueT>
{
  private var value: valueT?
  
  var attributes: [Attribute]
  
  var projectedValue: [Attribute]
  {
    return attributes
  }
  
  var wrappedValue: valueT?
  {
    get
    {
      return value
    }
    set
    {
      value = newValue
    }
  }
  
  init()
  {
    attributes = []
  }
  
  init(wrappedValue: valueT?)
  {
    self.init()
    
    self.value = wrappedValue
  }
  
  init(wrappedValue: valueT?, attributes: [Attribute])
  {
    self.init(wrappedValue: wrappedValue)
    
    self.attributes = attributes
  }
}
```

It is usual to provide three initialisers to a properrty wrapper that holds onto something other than the wrapped value:

```
  @Attributes()
```

… for when we don't want to pass anything to the additional storage and we want the wrapped value to be initilaised to its default value

```
  @Attributes(wrappedValue: nil)
```

… for when we don't want to pass anything to the additional storage but we do want to be able to initialise the wrapped value

```
  @Attributes(wrappedValue: nil, attributes: [])
```

… for when we want to pass in something to the additional storage as well as initialising the wrapped value.
  

Here we want to store a list of Attributes for a given property, so the property wrapper now gets storage to hold onto that list of attributes, along with a `projectedValue` property that allows us to access that list of attributes via the `@myProperty` syntax. Hence, the only initialiser it makes sense to use will be the last one.

### Defining Attributes

Here are two sample attributes that might be useful:

```
enum PersistencyAttribute : Attribute
{
  case persistent
  case transient
}

typealias ValidationClosure<valueT> = (valueT) -> Bool

struct ValidationAttribute<valueT> : Attribute
{
  let closure: ValidationClosure<valueT>
}
```

The first attribute is intended to be used by a persistence mechanism and simply marks the property with an enum value, detemining if the property is to be persisted or ignored.

The second attribute provides a mechanism for holding a reference to a closure that can be used to validate the property's value according to some rule.

### Defining a Validation

In this article, we will use a simple struct with a static method to act as the validation:

```
struct PersonValidation
{
  static var firstNameValidationClosure: ValidationClosure<String?>
  {
    return { $0 != nil }
  }
}
```

All this closure does is to check that the value is valid by ensuring it is not nil. It could do much more, as long as the end result is a boolean.

### Adding the Attributes to a Type

Let's start with a Person struct:

```
struct Person
{
  @Attributes(attributes: [PersistencyAttribute.persistent,
                           ValidationAttribute<String?>(closure: PersonValidation.firstNameValidationClosure)])
  var firstName: String? = nil
  
  @Attributes(attributes: [PersistencyAttribute.transient])
  var lastName: String? = nil
}
```

This is hardly a realistic example but it does succinctly demonstrate how different (possibly multiple) attributes can be applied to properties.

The least obvious 

### Making use of the Attributes

Let's start with the Persistency attribute:

```
{
  if let lastNamePersistencyAttribute = person.$lastName.first(where: { $0 is PersistencyAttribute} )
  {
    print("lastName : \(lastNamePersistencyAttribute)") // lastName : transient
  }

  if let firstNamePersistencyAttribute = person.$firstName.first(where: { $0 is PersistencyAttribute} )
  {
    print("firstName : \(firstNamePersistencyAttribute)") // firstName : persistent
  }
}
```

We could make life a bit easier by using a dictionary with an enum key for the type of the attribute but this code allows you to see what is going on under the hood a bit more clearly.

However, the above code is somewhat verbose and so, for the Validation attribute, we can wrap it inside an abstract protocol that can be implemented by the type that holds the property:

```
protocol Validatable
{
  func validate() -> Bool
}
```

Then we can implement this as an extension on the Person type:

```
extension Person : Validatable
{
  func validate() -> Bool
  {
    let attributes = $firstName
      
    if let validationAttribute = attributes.first(where: { $0 is ValidationAttribute<String?> }) as? ValidationAttribute<String?>
    {
      return validationAttribute.closure(firstName)
    }
    
    return false
  }
}
```

We start off by getting the projected value from the `firstName` property, using the `$` prefix. This returns the array of attributes, from which we then extract the Validation attribute. Then all that is left is to call the closure, passing in the current value of the property.

Multiple properties could thus be validated at the same time with only the one call to the Validate(_:) method on the instance.

Here's some sample test code:
```
{
  var person = Person()
  
  print(person.validate()) // false
  
  person.lastName = "`Jobs`"
  
  print(person.validate()) // false
  
  person.firstName = "Steve"
  
  print(person.validate()) // true
  
  person.firstName = nil
  
  print(person.validate()) // false
}
```

### Summary

This article has discussed how to create a mechanism that allows multiple attributes to be applied to a single property by means of a property wrapper that can contain a list of attributes.

The hope is taht Swift will evolve to a point where such workarounds are no longer necessary. You can contact me via Twitter on @JoannaMacDev with any reactions or comments. 
