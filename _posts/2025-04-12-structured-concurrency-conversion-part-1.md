---
layout: post
title: 'Structured Concurrency Conversion (Part 1)'
date: 2025-04-12 15:24 -0500
---

Introduced in [Swift 5.5](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md) way back in September of 2021, what I'm going to call "Structured Concurrency" is a mixture of [async/await](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0296-async-await.md), [Task](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md#task-api), and [Actors](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0306-actors.md). In short, though, it's the way to accomplish potentially long-running operations in a way that can be checked by the compiler in order to reduce (but not completely eliminate!) race conditions and corruption of data. 

For me, these new technologies has been a very difficult concept to grasp. New concepts to grasp being difficult is nothing new. I remember struggling with Swift after Objective-C being the only programming language I used on a daily basis ever. I remember struggling with SwiftUI after 10 years of using UIKit. The difference with these was that if you failed, it was easily visible but also the community was sort of failing and learning together in a relatively short amount of time. Additionally, there was no pressure to adapt either language as Objective-C interop was there from the beginning and SwiftUI adoption wasn't really feasible until recently just because so many APIs weren't at par with UIKit. If anything, with Swift 2 to 3 conversion being super painful, it actually benefitted you to sort of sit back and wait. 

Structured Concurrency is not similar to either Swift or SwiftUI becuase if you want to use Swift 6, odds are you're going to want to learn the techniques to make it correct otherwise you'll start getting warnings and errors in your code base. Whereas you can still write apps in Objective-C in UIKit, Apple is somewhat forcing us to adopt this model and if they don't someone else on your team might. 

So, what am I to do? My idea is to take a piece of Apple sample code and convert it over to something that uses async/await, Tasks, and Actors. I might not get it right but I seldomly do the first time and that's okay as long as I try. In this first post in a series, I'm going to talk about the code that is posted by Apple in order to give an overview of what's happening before I convert it. 

## Asynchronously Loading Images

The code from Apple is posted [here](https://developer.apple.com/documentation/uikit/asynchronously-loading-images-into-table-and-collection-views) and is from March of 2020. 

### Normal UIKit Setup

There is a `UICollectionViewController` and `UITableViewController` subclass each which utilize two bespoke mechansims to fetch images asynchronously. 

In each of these subclasses, they access the image within a diffable data source cell registration. Because it uses a diffable data source, it needs an object that is the basis of the snapshots. In this case, they call it `Item` and it looks like this:

```swift
class Item: Hashable {
    
    var image: UIImage!
    let url: URL!
    let identifier = UUID()
    
    func hash(into hasher: inout Hasher) {
        hasher.combine(identifier)
    }
    static func == (lhs: Item, rhs: Item) -> Bool {
        return lhs.identifier == rhs.identifier
    }
    
    init(image: UIImage, url: URL) {
        self.image = image
        self.url = url
    }

}
```

That's right, it's a class that has a `var image: UIImage!`. Within both the collection and table view controllers, they instantiates the `Item`s with a placeholder image which is initially shown. Later, we will asynchronously fetch the correct image at the URL and replacing that image. We'll be taking a look at the `UITableViewController` version that does that when it makes the cell registration for the data source.

```swift
dataSource = UITableViewDiffableDataSource<Section, Item>(tableView: tableView) {
    (tableView: UITableView, indexPath: IndexPath, item: Item) -> UITableViewCell? in
    let cell = tableView.dequeueReusableCell(withIdentifier: "cell", for: indexPath)
    /// - Tag: update
    var content = cell.defaultContentConfiguration()
    content.image = item.image
    ImageCache.publicCache.load(url: item.url as NSURL, item: item) { (fetchedItem, image) in
        if let img = image, img != fetchedItem.image {
            var updatedSnapshot = self.dataSource.snapshot()
            if let datasourceIndex = updatedSnapshot.indexOfItem(fetchedItem) {
                let item = self.imageObjects[datasourceIndex]
                item.image = img
                updatedSnapshot.reloadItems([item])
                self.dataSource.apply(updatedSnapshot, animatingDifferences: true)
            }
        }
    }
    cell.contentConfiguration = content
    return cell
}
```

Walking through this, the cell gets dequeued from the table view and we make a content configuration. It immediately sets the item's image to the content configuration's image property. This _could_ be the placeholder but, later, it could _also_ be the real image. At this point, even if it isn't the placeholder image, we still go and fetch the image at the URL. We'll talk about that block of code in just a minute. While that asynchronous work is being done, the cell's content configuration is set and we return the cell. 

In the closure, when the image comes back from an undetermined timeframe, we check to make sure the image is not `nil` and that the fetched image is **not** the image previously set. This seems inefficient since we are making a network call (or getting the cached data) no matter what but then again, we are also capturing `self` in the closure and not making it `weak` but pobody's nerfect[^1]. Moving on, we make a mutable snapshot, get the index of the object we want to alter and get a reference to a reference to the item which we are storing in an array as a property. (In my opinion, I don't think we should have two sources of truth with the diffable data source **and** this array.) We set the item's image and then tell the snapshot to reload the item and apply the snapshot. It will then run through this cell registration again and set the cell configuration's image to the updated image. 

### Image Cache

The `ImageCache` is an object that holds onto an `NSCache` with the key being the `NSURL` and the value being `UIImage`. The `ImageCache` also has a Dictionary where the key is `NSURL` again but the value is an Array of closures that has the arguments of `(Item, UIImage?)` and returns `Void`. They look like this: 

```swift
private let cachedImages = NSCache<NSURL, UIImage>()
private var loadingResponses = [NSURL: [(Item, UIImage?) -> Swift.Void]]()
```

There is a simple function for returning an optional image from the cache using the url. Not entirely sure why it's there and why it's `public` given the only caller is the `ImageCache` itself. 

The meat of the work is done in this big function:

```swift
final func load(url: NSURL, item: Item, completion: @escaping (Item, UIImage?) -> Swift.Void) {
    // Check for a cached image.
    if let cachedImage = image(url: url) {
        DispatchQueue.main.async {
            completion(item, cachedImage)
        }
        return
    }
    // In case there are more than one requestor for the image, we append their completion block.
    if loadingResponses[url] != nil {
        loadingResponses[url]?.append(completion)
        return
    } else {
        loadingResponses[url] = [completion]
    }
    // Go fetch the image.
    ImageURLProtocol.urlSession().dataTask(with: url as URL) { (data, response, error) in
        // Check for the error, then data and try to create the image.
        guard let responseData = data, let image = UIImage(data: responseData),
            let blocks = self.loadingResponses[url], error == nil else {
            DispatchQueue.main.async {
                completion(item, nil)
            }
            return
        }
        // Cache the image.
        self.cachedImages.setObject(image, forKey: url, cost: responseData.count)
        // Iterate over each requestor for the image and pass it back.
        for block in blocks {
            DispatchQueue.main.async {
                block(item, image)
            }
            return
        }
    }.resume()
}
```

Whew! The comments do a good job of explaing what's going on there but we check for the cached image and call the completion if it exists. Next up, they add the closure to the array for that URL in the dictionary and return OR create an entry in the dictionary for the URL and an array with the first completion. 

Moving on, we use the `ImageURLProtocol.urlSession()` (more on this later) data task with a completion that has the arguments of optional data, response, and error. We immediately resume that data task. When the data task is complete, the data task's closure gets executed. Again, no `[weak self]` here but we first check the data, that a `UIImage` can be created with the data, that there is an array of closures to call, and that the error is `nil` otherwise we call the completion with no image (on the main thread). With those pieces, we then set the image to the cache but also go through each completion closure and execute it with the image (on the main thread). 

### ImageURLProtocol? 

Did you know you can sort of override `URLSession` to have the same API but act differently? You need to create something that adheres to the [`URLProtocol`](https://developer.apple.com/documentation/foundation/urlprotocol). This is largely done in this protocol function:

```swift
final override func startLoading() {
    guard let reqURL = request.url, let urlClient = client else {
        return
    }
    
    block = DispatchWorkItem(block: {
        if self.cancelledOrComplete == false {
            let fileURL = URL(fileURLWithPath: reqURL.path)
            if let data = try? Data(contentsOf: fileURL) {
                urlClient.urlProtocol(self, didLoad: data)
                urlClient.urlProtocolDidFinishLoading(self)
            }
        }
        self.cancelledOrComplete = true
    })
    
    ImageURLProtocol.queue.asyncAfter(deadline: DispatchTime(uptimeNanoseconds: 500 * NSEC_PER_MSEC), execute: block)
}
```

What is happening here is that we make sure we have a url and client but then set up a closure that will get the data from the URL and call the protocol's functions signaling that the work is done[^2]. That closure is sent to a `DispatchSerialQueue` to be executed in 0.5 seconds. There is also a property (`cancelledOrComplete`) on the class that signifies that it is done. 

This `cancelledOrComplete` is used in case the data task is cancelled.

```swift
final override func stopLoading() {
    ImageURLProtocol.queue.async {
        if self.cancelledOrComplete == false, let cancelBlock = self.block {
            cancelBlock.cancel()
            self.cancelledOrComplete = true
        }
    }
}
```

This class also has a deprecated `OS_dispatch_queue_serial` which is renamed `DispatchSerialQueue`.

## In Conclusion

Okay, we have three pieces that need to be addressed. 

* We have our table/collection view call sites that have an asynchronous closure to update the model driving the diffable data source.
* We have the object that asynchronously returns the cached image or fetches the image data and manages all of the requests to do so.  
* We have an override for `URLSession` that gets the image off disk and returns it after a half second. 

In the following parts of this conversion, I'll be working from the bottom of this list up to the table/collection view. As a bonus, I'll be using this modern mechanism to drive a SwiftUI equivalent view. Up next is [part 2](/structured-concurrency-conversion-part-2)!

[^1] Normally, I would not dunk on someone else's code but Apple should really know better here.  
[^2] Apple missed a protocol function call of `func urlProtocol(_ protocol: URLProtocol, didReceive response: URLResponse, cacheStoragePolicy policy: URLCache.StoragePolicy)` before calling `func urlProtocol(_ protocol: URLProtocol, didLoad data: Data)`. With that missing, the app would crash when I later translate over to Structured Concurrency. Fun!