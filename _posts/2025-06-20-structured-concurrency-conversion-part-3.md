---
layout: post
title: Structured Concurrency Conversion (Part 3)
date: 2025-07-01 10:49 -0500
---

Alright, in [part 1](structured-concurrency-conversion-part-1), I laid out the code structure is for Apple's [Async Image Loading](https://developer.apple.com/documentation/uikit/asynchronously-loading-images-into-table-and-collection-views) sample code. In [part 2](structured-concurrency-conversion-part-2), I fixed up the Xcode project—something you'll need to do when starting a new project because Swift 6 and strict checking aren't on by default (as of June 2025). Today, we'll actually convert the dispatch and closure-based code to use Actors and async/await.

#### A Disclosure

Before we start, I want to reference back to Swift's release in 2014. There was a tendency for developers with years of Objective-C experience to write Swift code using the same patterns, which often resulted in the fuzzy, subjective judgment that the code wasn't "Swifty" enough. Something similar is happening again with Structured Concurrency. After years of writing code that explicitly manages threading and handles locks, we now have a handy tool that allows us to catch potential data races before compilation. However, as we learn this new approach, our code might initially mimic older patterns to some degree. The reason I mention this is if you squint at the old code in the exercise, you should be able to see how it translates to something more modern. The underlying patterns and structure are somewhat present, which might help you bridge from the old way of doing things to the new, even if it doesn't fully embrace all the new paradigm has to offer.

## `ImageCacheActor`

Let's begin by making an Actor called `ImageCacheActor`. 

### Properties

We'll have a [singleton static variable](https://github.com/jacobvanorder/StructuredConcurrencyAsynchronouslyLoadingImagesIntoTableAndCollectionViews/commit/6a24b8f1a45fbae6b13a2ea1e0110e0c5cff8e48#diff-5d11c3dfe5f4c8ab2ce5f1c202dc079762f709dbb4a243d9193489eb77b721d7R16) that can be accessed from various places just like the old version. 

The old version uses `NSCache<NSURL, UIImage>` to cache the images but is that `Sendable` or thread-safe? The documentation does say:

	> You can add, remove, and query items in the cache from different threads without having to lock the cache yourself.

Let's [roll with it](https://github.com/jacobvanorder/StructuredConcurrencyAsynchronouslyLoadingImagesIntoTableAndCollectionViews/commit/6a24b8f1a45fbae6b13a2ea1e0110e0c5cff8e48#diff-5d11c3dfe5f4c8ab2ce5f1c202dc079762f709dbb4a243d9193489eb77b721d7R21-R22)! We are making it `@MainActor` because we will want to access it later from the Table/CollectionView in order to determine if we even _need_ to fetch the image. This will need to be done from the Main Actor.

 The next and final property is a dictionary that used to contain a Dictionary where the key was the `NSURL` and the value was `[(Item, UIImage?) -> Swift.Void]` *(Note, that's an Array of closures)*. The purpose of this Dictionary is for the case the image was already loading but a newly dequeued cell requested the image again which would pass in a new completion closure. When the image is loaded, it will call the original closure but also all subsequent closures if other callers requested the same URL. 

 What we'll be doing is converting that to a Dictionary where the key is still `NSURL` but the value will be the **first** task ([See here](https://github.com/jacobvanorder/StructuredConcurrencyAsynchronouslyLoadingImagesIntoTableAndCollectionViews/commit/6a24b8f1a45fbae6b13a2ea1e0110e0c5cff8e48#diff-5d11c3dfe5f4c8ab2ce5f1c202dc079762f709dbb4a243d9193489eb77b721d7R25)). **This is the first big shift in thinking because of the top-down approach of async/await.** We'll get into what that means in the implementation described down below.

 Okay, the properties are out of the way so let's get to the meat of it. 

 ### Functions

 Our first function follows what we had before with a public interface for our `NSCache`. This works, though, because well, _"You can add, remove, and query items in the cache from different threads without having to lock the cache yourself."_. If you say so! We are annotating it as `@MainActor` for reasoned explained above.

 #### Load URL

 The real work gets done in `final func load(url: URL) async throws -> UIImage`. You'll notice that we changed the [signature](https://github.com/jacobvanorder/StructuredConcurrencyAsynchronouslyLoadingImagesIntoTableAndCollectionViews/commit/6a24b8f1a45fbae6b13a2ea1e0110e0c5cff8e48#diff-5d11c3dfe5f4c8ab2ce5f1c202dc079762f709dbb4a243d9193489eb77b721d7R34) to take in a `URL` but no `Item` as I don't want to tie this utilitarian functionality to a specific model object type. Also, we change the completion handler to something that is `async`, `throws` an error, and returns `UIImage`. Before, the function gave no indication that something went wrong, it just returned `nil` for the image which is better than returning the placeholder image, I guess. 

 Let's get to the meat of the function!

I'm actually going to defer my explaination of the `defer` to the end. Moving on…

 After the `defer`, the first step is to check the cache using the function outlined above and returned the cached image if there is one.

```swift
        // Check for a cached image.
        if let cachedImage = await image(url: url) {
            return cachedImage
        }
```
We need to `await` because we are changing contexts between our Actor and the `@MainActor`. Not a huge deal as `NSCache` is safe and our function is `async` anyway.

 Remember that Dictionary where the key was the `NSURL` and the value was a `Task`? Time to shine `loadingResponses`!

 ```swift
        // In case there are more than one requestor for the image, we wait for the previous request and
        // return the image (or throw)
        if let previousTask = loadingResponses[url] {
            return try await previousTask.value
        }
```

What this code does is check for a previous request (we will discuss shortly) in a `Task` at that `URL` which is stored in a Dictionary. If we did make a request earlier, we will tell whatever subsequent caller that is loading the image at that `URL` to hold on for the result of that first call. _This is quite the shift in thinking!_
Previously, we [held each request's completion handler in an array](https://github.com/jacobvanorder/StructuredConcurrencyAsynchronouslyLoadingImagesIntoTableAndCollectionViews/commit/5b0169831551202154c78c20da0e81cde67f3ee2#diff-5d11c3dfe5f4c8ab2ce5f1c202dc079762f709dbb4a243d9193489eb77b721d7R30-R35) and then [iterate over each completion closure stored](https://github.com/jacobvanorder/StructuredConcurrencyAsynchronouslyLoadingImagesIntoTableAndCollectionViews/commit/5b0169831551202154c78c20da0e81cde67f3ee2#diff-5d11c3dfe5f4c8ab2ce5f1c202dc079762f709dbb4a243d9193489eb77b721d7R48-R54) once the image comes back and is valid. 

Next up, we make a `Task<(UIImage), any Error>` and save it to a local `let` variable. 

```swift
        // Go fetch the image.
        let currentTask = Task {
            let (data, _) = try await ImageURLAsyncProtocol.urlSession().data(from: url)
            // Try to create the image. If not, throw bad image data error.
            guard let image = UIImage(data: data) else {
                throw LoadingError.badImageData
            }
            // Cache the image.
            await setCachedImage(image, atUrl: url)
            return image
        }
```

In the task, we asynchronously fetch the data from the `URL`. When that is done, we make sure it's a valid `UIImage` or `throw` an error. If it is valid, we use a _function_ (more on that in a second) to set the image to the cache, and, finally, return the image as the `Task`'s value. 

This is the part of the code that lines up well with the old way of fetching `URLSession` data, getting a completion closure, and handling the result. In fact, it lines up so well, it's probably doesn't go far enough to transform the code to adjust to the new way of working which [I apologized for before](#a-disclosure). Much like Apple probably looks at their original code sample and might cringe, so will I in five years.

After that `Task` is made, we store it in the `loadingResponses` Dictionary for the `URL` and then asynchronously return the eventual value of the task or throwing a possible error.  

Back to the `defer` at top of the function. This one:

```swift
	defer { loadingResponses.removeValue(forKey: url) }
```

If you think about it, we have a shared Singleton of `ImageCacheActor` which has a `NSCache` and a `Dictionary<URL: Task<(UIImage), any Error>]`. That `Task` will hold on to the value as long as we keep it around. In essence, it could be our own cache if we wanted it to but, `NSCache` has some nice features such as flushing memory, if needed, that we get for free. In order to hold less memory, let's remove the `Task` from the dictionary and it will get freed up. 

This is the end of our big `func load(url: URL) async throws -> UIImage` function!

_But why a function for setting the image to the cache?_

If you try to set the object directly on the `NSCache` via `cachedImages.setObject(image, forKey: url as NSURL)`, you will get the helpful message of `Non-sendable type 'NSCache<NSURL, UIImage>' of property 'cachedImages' cannot exit main actor-isolated context; this is an error in the Swift 6 language mode`. Calling with an `await` doesn't matter. This is why we come up with [this function](https://github.com/jacobvanorder/StructuredConcurrencyAsynchronouslyLoadingImagesIntoTableAndCollectionViews/commit/6a24b8f1a45fbae6b13a2ea1e0110e0c5cff8e48#diff-5d11c3dfe5f4c8ab2ce5f1c202dc079762f709dbb4a243d9193489eb77b721d7R63-R66):

```swift
    @MainActor
    private func setCachedImage(_ cachedImage: UIImage, atUrl url: URL) {
        cachedImages.setObject(cachedImage, forKey: url as NSURL)
    }
```

The property we are trying to alter is `@MainActor` so let's annotate this function the same. Once we do that, we can set the property directly in this function and cross the contexts by `await`ing when we call it. 

## Table/Collection View Implementation

You can look at the [diff](https://github.com/jacobvanorder/StructuredConcurrencyAsynchronouslyLoadingImagesIntoTableAndCollectionViews/commit/6a24b8f1a45fbae6b13a2ea1e0110e0c5cff8e48#diff-8036ee87cac1d8185a01c5994466a2f90fa62663e984965d58915ae176a4026dL25-R44) for this change but previously Apple set the image no matter of what it was and, even if the image was loaded, it would fetch the image no matter what. Now, this isn't a big deal because we pull from the cache but it seems unnecessary. Now, we have a three step process: 

1. [Does our model object have it? Use that](https://github.com/jacobvanorder/StructuredConcurrencyAsynchronouslyLoadingImagesIntoTableAndCollectionViews/commit/6a24b8f1a45fbae6b13a2ea1e0110e0c5cff8e48#diff-8036ee87cac1d8185a01c5994466a2f90fa62663e984965d58915ae176a4026dR26-R27).
2. Does the shared cache have it in their `NSCache`? [Use that](https://github.com/jacobvanorder/StructuredConcurrencyAsynchronouslyLoadingImagesIntoTableAndCollectionViews/commit/6a24b8f1a45fbae6b13a2ea1e0110e0c5cff8e48#diff-8036ee87cac1d8185a01c5994466a2f90fa62663e984965d58915ae176a4026dR28-R29).
3. Set a placeholder image and go fetch the image, [like so](https://github.com/jacobvanorder/StructuredConcurrencyAsynchronouslyLoadingImagesIntoTableAndCollectionViews/commit/6a24b8f1a45fbae6b13a2ea1e0110e0c5cff8e48#diff-8036ee87cac1d8185a01c5994466a2f90fa62663e984965d58915ae176a4026dR31-R32).

In order to fetch the image, we create a `Task` where we use our new `ImageCacheActor` to load the image from the `URL` asynchronously. If an error is thrown, we now set a broken image. Then it is a matter of setting the image to the `Item` that is used to drive the diffable data source and then asynchronously apply the updated snapshot. The cell will reload and it will use the first scenario of the model object's image. 

## In Conclusion

That was a massive change!

We have some optimizations done here where we are no longer holding onto an array of completion handlers which themselves held a strong reference to the collection views. Additionally, we do not load the cached version of the image even though the item has loaded it and it is stored in memory in the `Item`. 

From a code organization standpoint, we had to shift some blocks of the code around now that we were no longer capturing functionality and storing it for later. Instead, if we have a loading task in progress, we ask subsequent requesters to wait for the first call. 

Coming up in Part 4, we'll be focusing on that URLProtocol subclass.