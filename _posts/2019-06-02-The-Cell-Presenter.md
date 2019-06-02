---
layout: post
title: The Cell Presenter
---

### Introduction

How many times have you written an iOS app, containing a UITableView or UICollectionView, where you end up putting all the cell configuration and updating code in the cell class itself; all the outlets and actions from components in the cell are connected to the cell whereas, with a UIView, they would be connected to a view controller.

So began the quest for a mechanism that would allow us to create a presenter (or controller) that would be responsible for automatically managing the visual state of a cell in response to changes in its model, as well as responding to interaction with the cell's components by updating the cell's model. 

### The Example Model

As is customary with so much example code, we shall use a Person class as the Model to be represented in a custom cell :

```
class Person : NotifyPropertyChanged
{
  var name: String
  {
    didSet
    {
      onPropertyChanged(\Person.name)
    }
  }
  
  var age: Int
  {
    didSet
    {
      onPropertyChanged(\Person.age)
    }
  }
  
  init(name: String, age: Int)
  {
    self.name = name
    
    self.age = age
  }
  
  lazy var propertyChanged: PropertyChangedEvent<Person> = .init(sender: self)
}


extension Person : Equatable
{
  static func == (lhs: Person, rhs: Person) -> Bool
  {
    return lhs === rhs
  }
}
```

As you can see, the Person class also implements the `NotifyPropertyChanged` protocol we discussed in an earlier article. This allows us to notify the cell presenter when changes to the model are made, whether that be from within the cell, or from elsewhere.

### Cell Requirements

One problem with Interface Builder is that cell reuse identifiers are entered as strings and the temptation is to use those strings in other code. One thing we can do is to standardise the reuse identifier on the cell class's type and this can be simply achieved by creating a protocol that encapsulates that behaviour :

```
public protocol IdentifiableCell { }

extension IdentifiableCell
{
  public static var reuseIdentifier: String
  {
    return String(describing: self)
  }
}
```

This protocol can be adopted by our custom cell, automatically giving the cell its unique identifier. all that's needed is to add `IdentifiableCell` to the cell type's declaration.

Whilst we are doing that, we might as well specify the designed height of the cell, for later use in the class that implements the UITableViewDelegate protocol :

```
public class PersonCell : UITableViewCell, IdentifiableCell
{
  public static let rowHeight: CGFloat = 100
  
  @IBOutlet weak var nameLabel: UILabel!

  @IBOutlet weak var ageLabel: UILabel!

  var buttonTapClosure: (() -> ())?

  @IBAction func btnTap()
  {
    buttonTapClosure?()
  }
}
```

This particular cell contains two labels to display the name and age of the Person, as well as a button that we will use to change the age of the corresponding Person (some might say cruelly) to a random number between 20 and 100.

Of course, there are much more useful things we could do, but this will serve as a simple exercise.

### The Base Presenter Class

Following the principles of the Model View Presnter design pattern, we will now need a base presenter class to link the view (cell) to its model :

```
open class Presenter<modelT, viewT : UIView> : NSObject
{
  public var model: modelT
  {
    didSet
    {
      updateView()
    }
  }
  
  public let view: viewT
  
  public required init(with view: viewT, for model: modelT)
  {
    self.view = view
    
    self.model = model
    
    updateView()
  }
  
  open func updateView() { }
}
```

This is then further extended to specialise it to our specific model end cell types :

```
class PersonCellPresenter : Presenter<Person, PersonCell>, PropertyChangeHandler
{
  public override func updateView()
  {
    // connect model and view to this presenter
    model.propertyChanged += propertyChangeClosure
    
    view.buttonTapClosure =
    {
      self.model.age = Int.random(in: 20...100)
    }
    
    // initialise view to reflect the model's current state
    DispatchQueue.main.async
    {
      self.view.nameLabel.text = self.model.name
      
      self.view.ageLabel.text = String(self.model.age)
    }
  }
  
  lazy var propertyChangeClosure: PropertyChangedEvent<Person>.Closure = .init
  {
    sender, args in // (sender: Person, args: PropertyChangedEvent<Person>.Args)
    
    DispatchQueue.main.async
    {
      switch args.keyPath
      {
        case \Person.name:
          self.view.nameLabel.text = sender.name
        case \Person.age:
          self.view.ageLabel.text = String(sender.age)
        default:
          break
      }
    }
  }
}
```

You may have noticed that the cell responds to the button tap by calling an optional closure held in the var :

```
var buttonTapClosure: (() -> ())?
```

The presenter assigns its own closure to that var, thus enabling it to respond to the button tap here rather than in the cell.

### Removing the Data Source Code from the UITableViewController

Usually, the data handling code tends to get put in the `UITableViewController` subclass, simply because the `UITableViewDataSource` is implemented there by default. So we end up with boilerplate code like this in every table view controller we need to specialise to a given model.

```
  public func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int
  {
    return model.count
  }

  override func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell
  {
    let cell = tableView.dequeueReusableCell(withIdentifier: PersonCell.reuseIdentifier, for: indexPath) as! PersonCell
    
    let person = model[indexPath.row]
    
    cell.nameLabel.text = person.name
    
    cell.ageLabel.text = String(person.age)
    
    return cell
  }
```

This kind of boilerplate code is one of the major causes of Massive View Controller syndrome and it can be so easily avoided by separating the data handling code into a "sub-controller" (in this case)

So, let's start with a generic class that I will call an Interactor, in line with the nomenclature used in the Model View Presenter design pattern, to represent a "sub-controller" that handles one specific part of the overall presenter's work :

```
open class TableViewDataSourceInteractor<modelT : Equatable,
viewT : UITableViewCell & IdentifiableCell,
cellPresenterT : CellPresenter<modelT, viewT>> : NSObject, UITableViewDataSource
{
  public var model: [modelT]
  
  public lazy var cellPresenterCache: CellPresenterCache<modelT, viewT, cellPresenterT> = .init()
  
  public init(model: [modelT])
  {
    self.model = model
  }
  
  public func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int
  {
    return model.count
  }
  
  public func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell
  {
    let cell = tableView.dequeueReusableCell(withIdentifier: viewT.reuseIdentifier, for: indexPath) as! viewT

    let item: modelT = model[indexPath.row]

    self.cellPresenterCache.bind(view: cell, to: item)

    return cell
  }
}
```

To make life easier, we can specify a typealias adapted to our specific purpose for our table view controller :

```
typealias PersonTableViewDataSourceInteractor = TableViewDataSourceInteractor<Person, PersonCell, PersonCellPresenter>
```

All that's needed now is to make the table view controller aware of this helper interactor :

```
class ViewController: UITableViewController
{
  private lazy var model: [Person] =
  {
    var items: [Person] = .init()
    
    items.append(Person(name: "Steve Jobs", age: 21))
    
    items.append(Person(name: "Tim Cook", age: 23))
    
    return items
  }()

  private lazy var dataSourceInteractor: PersonTableViewDataSourceInteractor = .init(model: self.model)

  override func viewDidLoad()
  {
    super.viewDidLoad()
    
    tableView.dataSource = dataSourceInteractor
  }
  
  …
}
```

### How to Link Cells to Their Respective Presenters?

You may have noticed a couple of innocuous lines, hiding in the code for the generic `TableViewDataSourceInteractor` class :

```
  …
  
  public lazy var cellPresenterCache: CellPresenterCache<modelT, viewT, cellPresenterT> = .init()
  
  …
  
    self.cellPresenterCache.bind(view: cell, to: item)
    
  …
```

And here lies the solution to a problem that had puzzled me for some time.

If, for example, our table view is capable of showing ten cells at a time, then the table view manages a cache of around twelve cells, that will be queued once they have dropped out of the table view; hence the reuse identifier to allow us to recover the cached cells when we move beyond the index paths that were used for the initial display.

Now, it's all very well and good for the table view to take care of cell recycling but, if we want to create a presenter for each cell that's likely to be created or dequeued, then we could end up creating one presenter for every item in the model, rather than just for each of the recycled twelve cells

Thus the idea of a cache of presenters that can be "dequeued" in the same way as the cells.

Once again, a typealias helps to make the generic code simpler :

```
public typealias CellPresenter<itemT, cellT : UITableViewCell> = Presenter<itemT, cellT>
```

```
public class CellPresenterCache<itemT, viewT : UITableViewCell, cellPresenterT : CellPresenter<itemT, viewT>>
{
  lazy var presenters: [viewT : cellPresenterT] = .init()
  
  public init() { }
  
  public func bind(view: viewT, to item: itemT)
  {
    guard let presenter = presenters[view] else
    {
      let presenter = cellPresenterT(with: view, for: item)
      
      presenters[view] = presenter
      
      return
    }
    
    presenter.model = item
  }
}
```

What is happening here is that we are maintaining a dictionary of presenters, keyed on a cell.

Every time the table view retrieves a dequeued cell, it is passed to this cache, where we check whether there is already a presenter for that cell in the dictionary. If not, a new presenter is created, bound to its appropriate model and cell, then added to the dictionary; otherwise an existing presenter will be retrieved and its model will be reassigned to the item at the required index of the model.

So, there we have it, the CellPresenter, a mechanism for allowing us to manage interactions between items in a list and the cell that represent them, without having to put the code in either the table view controller or the cells.

Once the generic code is written and placed in a framework, the only "custom" code required for each specific table view, is for the model, the view (cell) and the presenter.

### N.B.

I have optimised the NotifyPropertyChanged mechanism, used in this article; I plan on writing a further article on that later.

### Summary

This article has discussed how to separate out the `UITableViewDataSource` delegate from a `UITableViewController`, thus reducing the code bloat often associated with many view controllers. It also allows us to place a `UITableView` on a `UIViewController` should we need to place a table view only on part of the screen.

It also discussed the creation of a cache of cell presenters that can be recycled along with the cell recycling managed by the table view.
