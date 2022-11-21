---
layout: post
title: UICollectionView in 2022
date: 2022-11-21 14:51 -0600
---

## Backstory

I was there at WWDC 12 when they announced collection views and the crowd was simultaneously impressed that they wrote such a flexible user interface element that was similar to the extensively used table view. But, that was **ten years ago**.

Since then, Apple has worked to smooth away some of the larger issues that have cropped up in the countless apps that have used a collection to show a list/grid on the screen, mainly:

* Autolayout and dynamic content is now a thing and static, predefined sizes aren't really relevant anymore.
* There are a slew of crashes related maintaining a source of truth behind the list/grid. 

To answer these issues, Apple introduced `UICollectionViewCompositionalLayout` and `UICollectionViewDiffableDataSource`. 

## Lately…

I've had to set up a couple `UICollectionView`s using these two tools and I've learned a couple patterns that make organization of the code such that I can keep track of what's important. Here are a couple of tips that I find using a `UICollectionView` much easier.

Here is the collection view I made as the example:

![collection view](/assets/images/CollectionView2022/collectionview2022.png)

You can download the sample code [here](https://github.com/jacobvanorder/CollectionViews-in-2022) to see how it comes together. 

### typealias

Modern collection views utilize Swift Generics and that's great! It makes it automatically cast what type of model or view that you're using when defining behavior. The problem is that littered throughout your code is `UICollectionView.CellRegistration<Cell, Item>` or `UICollectionViewDiffableDataSource<Section, Item>` which is verbose and can get repetitive. As a result, at the top of the file, I'll get this out of the way:

{% highlight swift %}

    typealias DiffableDataSource = UICollectionViewDiffableDataSource<Section, Item>
    typealias Snapshot = NSDiffableDataSourceSnapshot<Section, Item>
    typealias ImageCellRegistration = UICollectionView.CellRegistration<SymbolImageCell, Item>
    typealias TextCellRegistration = UICollectionView.CellRegistration<SymbolTextCell, Item>
    typealias HeaderRegistration = UICollectionView.SupplementaryRegistration<SymbolHeader>
    typealias CellProvider = DiffableDataSource.CellProvider
    typealias HeaderProvider = DiffableDataSource.SupplementaryViewProvider

{% endhighlight %}

In usage, I can just write:

{% highlight swift %}

    let imageCellRegistration = ImageCellRegistration { cell, indexPath, item in
        cell.loadItem(item)
    }

{% endhighlight %}

Instead of:

{% highlight swift %}

    let imageCellRegistration = UICollectionView.CellRegistration<SymbolImageCell, Item> { cell, indexPath, item in
        cell.loadItem(item)
    }

{% endhighlight %}

One goes off the page and the other doesn't but they still work the same and convey usage. 

### Section and Item

Right off the bat, I define the sections that the compose the collection view along with a struct that holds the underlying objects.

#### Section

In the past, you might have an `enum` that is backed by an integer that gives and idea of what you might be displaying. This makes it easier to suss out what `indexPath.section` _might_ be, right? After all, it is more clear that `indexPath.section == Section.Horizontal.rawValue()` is for the horizontal section than `indexPath.section == 0`, right? But you still get into a situation where you might not _want_ to show the section and it becomes an exercise in maintaining what is expected versus what is really there with somthing like `indexPath.section == Section.Horizontal.rawValue() && horizontalIsShowing()`. This gets ever more complicated if things show up in a different order or if you have more than two sections. 

Here are two things I do to do away with that. First, define just a plain enum that conforms to `Hashable`.

{% highlight swift %}

    enum Section: Hashable {
        case Horizontal
        case Vertical
    }

{% endhighlight %}

Then, when you need to know what section you're dealing with, `UICollectionViewDiffableDataSource` conveniently has `@MainActor func sectionIdentifier(for index: Int) -> SectionIdentifierType?`.

In usage, this means that when the data source or compositional layout need to know what cell or layout, respectively, to give for each section, it's no longer tied to a particular `indexPath.section` value. In code, it is as easy as: 

{% highlight swift %}

    guard let diffableDataSource = collectionView.dataSource as? DiffableDataSource,
          let section = diffableDataSource.sectionIdentifier(for: indexPath.section) else { return nil }

{% endhighlight %}

Note, that _section_ might not even be there. The data source could be empty and there might not be one but modern collection view APIs **allow for `nil` to be returned**. That being said, you can now use that `section` as an enum with a `switch` statement.

{% highlight swift %}

    switch section {
    case .Horizontal:
        return collectionView.dequeueConfiguredReusableCell(using: imageCellRegistration, for: indexPath, item: item)
    case .Vertical:
        return collectionView.dequeueConfiguredReusableCell(using: textCellRegistration, for: indexPath, item: item)
    }

{% endhighlight %}
 
#### Item

Diffable Data Sources really want homogenous types within them. From watching WWDC videos, it seems like Apple prefers that item identifier to be the unique identifier for your backing model object. In theory, this might be the same identifier that your server uses for your object, right? What I have encountered is that if that identifier is the same despite the object being different, you get into trouble. In other words, if you have an `Author` with `id` that is `1` and a `Book` with `id` that is `1`, the diffable data source is going to complain. 

As a result, I tend to set up a `struct` that has an `id` which is an `UUID`. 

{% highlight swift %}

    struct Item: Hashable {
        let id: UUID = UUID()
        let symbol: String
    }

{% endhighlight %}

No likely chance of collision there. I will then store my object directly in that struct. This might go against what Apple suggests but now I don't have to have a corresponding collection to look up whenever I need to access it, it is right there and I don't have two things to keep track of. Usage:

{% highlight swift %}

    let textCellRegistration = TextCellRegistration { cell, indexPath, item in
        cell.loadItem(item)
    }

{% endhighlight %}

The `Item` is simply a wrapper with a `UUID`. 

### Cells Define Layout

To recap, a compositional layout is comprised of a _section_ that may use a _group_ that may use one or more _items_. 

My last tip to when defining the compositional layout, I tend to move what I can to describe the presentation to the cell itself for reuse. Because I may want to present the cell elsewhere, I move the creation of any one of those elements (_section_, _group_, _item_) to the cell for use elsewhere. After all, the cell probably knows how it wants to be presented, right? 

In usage:

{% highlight swift %}

    let item: NSCollectionLayoutItem
    if sectionIdentifier == .Horizontal {
        item = SymbolImageCell.compositionLayoutItem()
    } else { // .Vertical
        item = SymbolTextCell.compositionLayoutItem(withinEnvironment: environment)
    }
    let group = NSCollectionLayoutGroup.horizontal(layoutSize: item.layoutSize, subitems: [item])
    let section = NSCollectionLayoutSection(group: group)
    section.boundarySupplementaryItems = [self.headerItem(forEnvironment: environment)]
    section.orthogonalScrollingBehavior = sectionIdentifier == .Horizontal ? .continuousGroupLeadingBoundary : .none
     return section

{% endhighlight %}

See how the items are defined by the cell itself? That code can be utilized wherever the cell is presented which I've had to do before. 

## In Summary

I'm happy to see that many of the rough edges from collection views have been resolved. Many issues with updating the collections backing them have been resolved and layout has been consolidated into a sane, logical place. 

If I missed anything or if you have your way of handling collection views you like, [feel free to reach out](mailto:jacob@sushigrass.com). 






