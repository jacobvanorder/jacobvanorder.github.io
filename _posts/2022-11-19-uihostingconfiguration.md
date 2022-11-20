---
layout: post
title: UIHostingConfiguration
date: 2022-11-19 19:09 -0600
---

# SwiftUI in Cells

Since SwiftUI was introduced, I've found myself wading into the new way of working and then quickly running back as soon as I encounter the need to navigate anywhere within the app using the framework's confusing and limited navigation paradigm. It works great if you're tapping on a cell and pushing to a detail view but anything greater, let's say a multi-step process with diverse model types, and it falls apart. 

But, this is way of the future! So, while Apple tries to iron out that whole ball of wax, what's the easiest, low-cost way to try SwiftUI out? 

I have found that in iOS 16, using `UIHostingConfiguration` on `UICollectionViewCell` or `UITableViewCell` provides a nice, light way to do in less lines of code what `UIKit` can achieve. 

Here's a clear-cut example of how to use it.

{% highlight swift %}

class SymbolCell: UICollectionViewCell {  // 1
    func load(symbolNameString: String, state: UICellConfigurationState) {  // 2
        self.contentConfiguration = UIHostingConfiguration(content: { // 3
            HStack(spacing: 12.0) {
                Image(systemName: symbolNameString)
                    .resizable()
                    .scaledToFit()
                    .frame(width: 44.0, height: 44.0)
                    .foregroundColor(state.isSelected ? .red : .black)
                Text(symbolNameString)
                    .font(.body)
                    .foregroundColor(state.isSelected ? .red : .black)
                Spacer()
            }
        })
    }
}

{% endhighlight %}

I'll outline the various points here:

1. I have a subclass of `UICollectionViewCell`. Pretty standard stuff.
2. I like to load my cells with a model object, in this case a string, with a function that I call when I dequeue the cell or set up the cell provider.
3. Here is where the magic happens. I set the cell's content configuration using `UIHostingConfiguration` with the initializer that takes in content. It is here that I can easily set up an image, text, and align left. 

And that's it! No autolayout or setting up state. 

The result is

![normal example](/assets/images/UIHostingConfiguration/uihostingconfiguration-normal.png)

### Updating

But what if you need to update the cell? 

First move the creation of the content into it's own function:

{% highlight swift %}

    @ViewBuilder static func createCellContent(symbolNameString: String, state: UICellConfigurationState) -> some View {
        HStack(spacing: 12.0) {
            Image(systemName: symbolNameString)
                .resizable()
                .scaledToFit()
                .frame(width: 44.0, height: 44.0)
                .foregroundColor(state.isSelected ? .red : .black)
            Text(symbolNameString)
                .font(.body)
                .foregroundColor(state.isSelected ? .red : .black)
            Spacer()
        }
    }

{% endhighlight %}

In the original function to load the model, you can utilize that both for the initial content configuration but also the cell's `configurationUpdateHandler`. The result for the function that loads the model object looks like this:

{% highlight swift %}

    func load(symbolNameString: String, state: UICellConfigurationState) {
        self.contentConfiguration = UIHostingConfiguration(content: { SymbolCell.createCellContent(symbolNameString: symbolNameString, state: state)})
        
        self.configurationUpdateHandler = {(cell, updatedState) in
            cell.contentConfiguration = UIHostingConfiguration(content: { SymbolCell.createCellContent(symbolNameString: symbolNameString, state: updatedState) })
        }
    }

{% endhighlight %}

Now, when you tap on the cell, you can see a state change: 

![selected example](/assets/images/UIHostingConfiguration/uihostingconfiguration-selected.png)


To see this in action, check out the repo [here](https://github.com/jacobvanorder/UIHostingConfiguration).