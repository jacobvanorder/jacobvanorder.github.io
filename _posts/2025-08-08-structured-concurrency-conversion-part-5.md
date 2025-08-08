---
layout: post
title: Structured Concurrency Conversion (Part 5)
date: 2025-08-08 11:52 -0500
---

Part 5! Just as a recap: [part 1](/structured-concurrency-conversion-part-1), I laid out the code structure for Apple's [Async Image Loading](https://developer.apple.com/documentation/uikit/asynchronously-loading-images-into-table-and-collection-views) sample code. In [part 2](/structured-concurrency-conversion-part-2), I fixed up the Xcode project—something you'll need to do when starting a new project because Swift 6 and strict checking aren't on by default (as of June 2025). Today, we'll actually convert the dispatch and closure-based code to use Actors and async/await. In [part 3](/structured-concurrency-conversion-part-3) we converted the `ImageCache`, responsible for fetching and caching image data to an `Actor` in order to safely use and mutate across contexts. [Part 4](/structured-concurrency-conversion-part-4) featured me getting rid of the `URLProtocol` subclass in order to utilize the Repository Pattern in order to chose between mock and live data being vended using Structured Concurrency. 

## SwiftUI

Introduced in 2019, SwiftUI is the "best way to build apps for Apple platforms" [according](https://developer.apple.com/news/?id=zqzlvxlm) to Apple. So, let's utilize our new, modern version of the image loading in SwiftUI. 

I merged the commit into `main` and you can follow along [here](https://github.com/jacobvanorder/StructuredConcurrencyAsynchronouslyLoadingImagesIntoTableAndCollectionViews/blob/main/AsynchronouslyLoadingImagesIntoTableAndCollectionViews/Async%20Image%20Loading/SwiftUIView.swift).

### Implementation

Because we want the image to only load when it is on screen, we will use a `LazyVGrid` with five columns. 

```swift
    var body: some View {
        ScrollView {
            LazyVGrid(columns: columns, spacing: 0) {
                ForEach(items) { item in
                    ItemView(item: item)
                }
            }
        }
    }
```

Nothing too wild here. Instead of using `AsyncImage`, let's write our own `ItemView` that uses our `ImageCacheActor`. 

```swift
    private struct ItemView: View {
        static let imageCacheActor: ImageCacheActor = ImageCacheActor.publicCache
        let item: Item
        @State private var image: UIImage?

        var body: some View {
            if let image {
                Image(uiImage: image)
                    .resizable()
                    .aspectRatio(1, contentMode: .fit)
            } else {
                Image(uiImage: ImageCacheActor.placeholderImage)
                    .resizable()
                    .aspectRatio(1, contentMode: .fill)
                    .scaleEffect(0.5)
                    .task {
                        do {
                            self.image = try await Self.imageCacheActor.load(imageAtURL: item.imageURL)
                        } catch {
                            self.image = UIImage(systemName: "wifi.slash")
                        }
                    }
            }
        }
    }
```

The code above is vastly easier to write than UIKit! Effectively, we have a static variable that all of the cells will use: our `publicCache` singleton. We have an `Item` to provide the URL for the image and then an optional `@State` variable to hold onto the image data if it is present. 

Within the body, we check to see if that image has been loaded and, if so, use it as is. If the image _is not loaded_ then we use our placeholder image and then use a `.task` modifier to fetch the image using our `imageCacheActor`. The `.task` modifier has two advantages: it allows for asynchronous code within _and_ has built-in cancelation if the view is no longer needed. 

### Easy Breezy

And that's it! In 42 lines of code, we have done something that took many lines in UIKit. Most importantly, we utlized the same mechanism for fetching the image that we used in UIKit with no modification which is a sign of a useful API. 

Some improvements could be injecting the `ImageCacheActor` so we aren't so tied to a particular fetching mechanism but I left it this was for the sake of simplicity. 

