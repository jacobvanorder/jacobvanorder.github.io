---
layout: post
title: Structured Concurrency Conversion (Part 4)
date: 2025-07-02 18:18 -0500
---

To catch you up, in [part 1](/structured-concurrency-conversion-part-1), I laid out the code structure for Apple's [Async Image Loading](https://developer.apple.com/documentation/uikit/asynchronously-loading-images-into-table-and-collection-views) sample code. In [part 2](/structured-concurrency-conversion-part-2), I fixed up the Xcode project—something you'll need to do when starting a new project because Swift 6 and strict checking aren't on by default (as of June 2025). Today, we'll actually convert the dispatch and closure-based code to use Actors and async/await. In [part 3](/structured-concurrency-conversion-part-3) we converted the `ImageCache`, responsible for fetching and caching image data to an `Actor` in order to safely use and mutate across contexts. 

Today, we are going to finish the last bit of infrastructure that will get the images for the Table/Collection Views. 

## iOS 2 was a Long Time Ago

[iPhoneOS 2.0](https://en.wikipedia.org/wiki/IPhone_OS_2) was released on July 11, 2008 which was 17 years ago. 

The mechanism that uses Apple's sample code to bypass the network and, instead, go to the bundle is [`URLProtocol`](https://developer.apple.com/documentation/foundation/urlprotocol). What's amazing to think about is that `URLProtocol` uses `URLSession` which came out five years later with iOS 7.0. Before that, you'd use [`NSURLConnection`](https://developer.apple.com/documentation/foundation/nsurlconnection). 

Needless to say, this does not support Swift Concurrency.  

## ImageURLProtocol

Let's do our best to change this class that uses `DispatchSerialQueue` into something that can use `Task`. You can follow along in [this commit](https://github.com/jacobvanorder/StructuredConcurrencyAsynchronouslyLoadingImagesIntoTableAndCollectionViews/commit/b7d43e50f9e338289f988c7b40068052d5dd125d).

In the `startLoading()` function, instead of creating a `DispatchWorkItem`, we will create a `Task` that we need to hold on in case we need to cancel it later in `stopLoading()`. Within this `Task`, we sleep for a random amount between 0.5 and 3.0 seconds. The rest of the code is basically the same with the only other difference being that we wrap it all in a `do/catch` in order to catch the errors, log them, and fail with an error. 

Job done! Right? Right?

### Those Damn Errors

```
❌ Passing closure as a 'sending' parameter risks causing data races between code in the current task and concurrent execution of the closure; this is an error in the Swift 6 language mode`
	ℹ️ Closure captures 'self' which is accessible to code in the current task
```

That's right, `self` is not safe to send between contexts because it is neither Sendable nor isolated. If you try to make it `Sendable` that's a no-no because this class inherits from `URLProtocol` and `'Sendable' class 'ImageURLAsyncProtocol' cannot inherit from another class other than 'NSObject'`. Besides, you have a property of the `Task` that is not safe to mutate from various contexts.  

You can't take `self` out of the equation because you need to send `self` to all of the `URLProtocolClient` functions you need to call in order to signal to the client that a response was received or if you failed. 

If we weren't using `self`, we could pass only the actual information, after making sure it was `Sendable`, by only capturing what you need in an [explicit capture list](https://forums.swift.org/t/swift-6-concurrency-error-of-passing-sending-closure/77048/3) but, again, `self` is being used. 

### `@preconcurrency` to the Rescue (?)

A former me would have thought, "Well, this API is definitely before Swift Concurrency.", let's mark it `@preconcurrency`. And that's what I did in this commit but it's not that simple. As [Matt Massicote points out](https://www.massicotte.org/preconcurrency), `@preconcurrency` _"alters how definitions are interpreted by the compiler. In general, it relaxes some rules that might otherwise make a definition difficult or even impossible to use."_. Notice he said _"relaxes some rules"_ and not _"fixes the underlying issues"_. 

The code Apple provided has a `static` `URLSession` that it uses to make network calls. [We use that](https://github.com/jacobvanorder/StructuredConcurrencyAsynchronouslyLoadingImagesIntoTableAndCollectionViews/commit/b7d43e50f9e338289f988c7b40068052d5dd125d#diff-5d11c3dfe5f4c8ab2ce5f1c202dc079762f709dbb4a243d9193489eb77b721d7R47-R49) to make network calls but we have no guarantee that the `ImageURLAsyncProtocol` will be unique despite me printing out `self` each time `startLoading()` is called and see:

```
<Async_Image_Loading.ImageURLAsyncProtocol: 0x600000c912c0>
<Async_Image_Loading.ImageURLAsyncProtocol: 0x600000c91380>
<Async_Image_Loading.ImageURLAsyncProtocol: 0x600000c17360>
<Async_Image_Loading.ImageURLAsyncProtocol: 0x600000c91410>
<Async_Image_Loading.ImageURLAsyncProtocol: 0x600000c91320>
<Async_Image_Loading.ImageURLAsyncProtocol: 0x600000c91200>
<Async_Image_Loading.ImageURLAsyncProtocol: 0x600000c12f10>
<Async_Image_Loading.ImageURLAsyncProtocol: 0x600000c19ec0>
```

But we can't _guarantee_ that. There is a very real chance that our class could be used multiple times to `startLoading()` and then `self` is captured in the `Task` and, before the request is fulfilled, another call to `startLoading()` is made and replaces the reference to `self`. Thus `self` does `risks causing data races between code in the current task and concurrent execution of the closure`. 

_FINE._

## An Alternative

What are we _really_ trying to do here? Apple is trying to show how to fetch data from _somewhere_ and they build in a delay to mimic an [EDGE](https://en.wikipedia.org/wiki/EDGE_(telecommunication)) connection. That _somewhere_ tips me off that we might be able to do something better. 

### Repository Pattern

Surprisingly, there is no wiki entry on the [Repository Pattern](https://www.geeksforgeeks.org/system-design/repository-design-pattern/) but it's been something I've been using since 2011 when it was introduced to me in a code base I was hired to update. 

Effectively, you provide an interface that dictates how you will provide data. You can then determine concrete functionality that will fulfill the promise of that interface in a specific way. For instance, you may want to provide your data from the network but you could also provide it from in-memory store or mock data in your bundle. Each one of those could be structures that adhere to the repository protocol but have distinct internal workings in order to provide the data. 

Back to our issue at hand, we have a need to fetch images from _somewhere_ and build in a delay. 

### The Interface

We will set up a protocol that gives the API of sorts to the consumer of what this provider will provide. Simply, it looks like [this](https://github.com/jacobvanorder/StructuredConcurrencyAsynchronouslyLoadingImagesIntoTableAndCollectionViews/blob/main/AsynchronouslyLoadingImagesIntoTableAndCollectionViews/Async%20Image%20Loading/ImageURLRepository.swift#L12-L14):


```swift
public protocol ImageURLRepository: Actor {
    func loadImage(atURL url: URL) async throws -> UIImage
}
```

You'll notice a couple things:

* We made whatever adheres to the protocol be an `Actor`
* It loads and image at a URL
* It is `async`, `throw`ing, and returns a `UIImage`

### Mock

Now that we have a contract of what we need to do, let's make our mock version. Thinking about what we'll want this mock provider to do, we'll want something that:

* Has a delay that could be random
* Goes to a `Bundle` and looks for the file name at the end of that `URL`
* Returns the image 

[Here](https://github.com/jacobvanorder/StructuredConcurrencyAsynchronouslyLoadingImagesIntoTableAndCollectionViews/blob/main/AsynchronouslyLoadingImagesIntoTableAndCollectionViews/Async%20Image%20Loading/ImageURLMockRepository.swift#L13-L31) is the code:

```swift
actor ImageURLMockRepository: ImageURLRepository {

    let delayRange: ClosedRange<Double>
    let bundle: Bundle

    func loadImage(atURL url: URL) async throws -> UIImage {
        try await Task.sleep(for: .randomSeconds(in: delayRange))
        
        let name = url.lastPathComponent
        guard let bundleURL = bundle.url(forResource: name, withExtension: "") else { throw ImageURLRepositoryError.imageDataNotFound }
        let data = try Data(contentsOf: bundleURL)
        return try Self.image(fromData: data)
    }

    init(delayedBetween start: Double, and end: Double, bundle: Bundle = .main) {
        self.delayRange = start...end
        self.bundle = bundle
    }
}
``` 

We have our delay range which we choose a random number from when we ask `Task` to `sleep` for a number of seconds. We also don't want to hard code our `Bundle` that we pull from as we might need a different one for testing or other scenarios. 

Then it's a matter of sleeping, pulling out the image name, getting the data, and returning the image.

### Network

With the same contract as the Mock source, let's make our network version. Thinking about what we'll want this network provider to do, we'll want something that:

* Takes in a `URLSession` to use for fetching
* Loads the data asynchronously

This is more clear-cut and the code looks like [this](https://github.com/jacobvanorder/StructuredConcurrencyAsynchronouslyLoadingImagesIntoTableAndCollectionViews/blob/main/AsynchronouslyLoadingImagesIntoTableAndCollectionViews/Async%20Image%20Loading/ImageURLNetworkRepository.swift#L12-L25):

```swift
actor ImageURLNetworkRepository: ImageURLRepository {

    let urlSession: URLSession

    func loadImage(atURL url: URL) async throws -> UIImage {
        let (data, response) = try await urlSession.data(from: url)
        guard (response as? HTTPURLResponse)?.statusCode != 404 else { throw ImageURLRepositoryError.imageDataNotFound }
        return try Self.image(fromData: data)
    }

    init(urlSession: URLSession = .shared) {
        self.urlSession = urlSession
    }
}
```

This is even less complicated because we aren't delaying. We fetch the data, check the response code, and return the image. You'll notice that we have a protocol extension for instantiating the `UIImage` from data. Both mock and network versions do that so I moved it into a [protocol extension](https://github.com/jacobvanorder/StructuredConcurrencyAsynchronouslyLoadingImagesIntoTableAndCollectionViews/blob/main/AsynchronouslyLoadingImagesIntoTableAndCollectionViews/Async%20Image%20Loading/ImageURLRepository.swift#L16-L21).

### Why an Actor?

Initially, if you were to get rid of the condition that the protocol must be `Actor` and you make the mock and network versions `final class`, initially, you'll get a `Sending 'self.repository' risks causing data races; this is an error in the Swift 6 language mode` error when you wrap `ImageCacheActor`'s usage in a `Task` [here](https://github.com/jacobvanorder/StructuredConcurrencyAsynchronouslyLoadingImagesIntoTableAndCollectionViews/blob/main/AsynchronouslyLoadingImagesIntoTableAndCollectionViews/Async%20Image%20Loading/ImageCacheActor.swift#L52). That might seem like a deal breaker but what it's mad about isn't `repository` but the `self.` part. To get around this, you capture only what you can add:

```
Task { [repository] in 
	…
}  
```

I found this when dealing with something similar with a SwiftUI's view and found [this](https://forums.swift.org/t/whats-the-best-way-to-resolve-the-concurrency-warning-capture-of-self-with-non-sendable-type-loadingview-in-a-sendable-closure/63631) Swift Forums post. Now in our case, it actually [might be a bug](https://forums.swift.org/t/task-capture-list-silence-data-race-error/78275) so, in order to be safe, I'll change my repositories back to an Actor. Additionally, I have been keeping some internal state in my repositories when using this pattern in production. Things like `next_page` or how many times something has been accessed. 

## In Conclusion

This is my final piece of the puzzle to change the Apple code over to something that is more modern. Ultimately, because `URLProtocol` is a byproduct of a different era of Apple development, it is too incompatible with Swift strict concurrency and a different approach needs to be made. 

So, that wraps this all up! If you see any issues or have any corrections, [please reach out](mailto:jacob@sushigrass.com). I'd like to send a very special thanks out to [Matt Massicotte](https://www.massicotte.org) for sanity-checking early versions of this series of blog posts. 

## What's Next?

I have some ideas about where to take this. How would I use the same code in SwiftUI? What impact do the changes in Swift 6.2 have towards this code base? Anything else you'd like to see? Reach out on [Mastodon](https://mastodon.social/@jacobvo).






