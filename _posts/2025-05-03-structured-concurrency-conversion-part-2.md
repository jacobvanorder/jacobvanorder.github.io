---
layout: post
title: Structured Concurrency Conversion (Part 2)
date: 2025-05-03 13:54 -0500
---

Alright, in [part 1](/structured-concurrency-conversion-part-1), I laid out the code structure is for Apple's [Async Image Loading](https://developer.apple.com/documentation/uikit/asynchronously-loading-images-into-table-and-collection-views) sample code. In this session, we'll update the settings of the project. 

## Updating the Project 

First thing is that we want not capture `self` in closures, add `func urlProtocol(_ protocol: URLProtocol, didReceive response: URLResponse, cacheStoragePolicy policy: URLCache.StoragePolicy)`, modernize the Xcode project settings, and, most importantly, turn on Swift 6 and `Strict Concurrency Checking` to `Complete`. 

![Xcode Project Settings](/assets/images/2025-05-03-structured-concurrency-conversion-part-2/Xcode_Project_Settings.png)

We also want to bump the minimum deployment target to iOS 18.0. 

## Two Bugs 

### `[weak self]`

When using completion closures and classes, it important to make sure you're not capturing a reference to `self` in the closure in case the closure is never called which would cause the strong reference to `self` to never be released. 

Apple does this in a [could places](https://github.com/jacobvanorder/StructuredConcurrencyAsynchronouslyLoadingImagesIntoTableAndCollectionViews/commit/7af3fbdcc3c6970468b3c073173abd34aa20a28d) and it's important to fix those even though we'll probably replacing the closures with async/await variants. 

### URLProtocol Did Receive Response

For some reason, when using Apple's code this does not crash but when changing to an Actor later, it _does_ crash. 

The [documentation](https://developer.apple.com/documentation/foundation/urlprotocol) is sparse and gives no indication what might be required. There is this [ancient code sample](https://developer.apple.com/library/archive/samplecode/CustomHTTPProtocol/Listings/Read_Me_About_CustomHTTPProtocol_txt.html#//apple_ref/doc/uid/DTS40013653-Read_Me_About_CustomHTTPProtocol_txt-DontLinkElementID_23) but it says that the authentication calls are also required but I'm not seeing that. but Stack Overflow [comes to the rescue](https://stackoverflow.com/a/76231740).

Well, let's fix that [here](https://github.com/jacobvanorder/StructuredConcurrencyAsynchronouslyLoadingImagesIntoTableAndCollectionViews/commit/0d1ff294ad1eafb0c7ac413aa21da878b8a42047#diff-046d5f20089704bd618489bfe91acefefb2cb2c114418b5e855fbfd9601ddc5bR38-R44). 

## Ready to Go

With those changes done, we are ready to make our modern changes so look for that in [part 3](/structured-concurrency-conversion-part-3) and [part 4](/structured-concurrency-conversion-part-4).