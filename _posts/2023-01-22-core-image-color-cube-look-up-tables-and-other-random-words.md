---
layout: post
title: Core Image, Color Cube, Look Up Tables, and Other Random Words
date: 2023-01-22 14:10 -0600
---

![An Example](/assets/images/2023-01-22-core-image-color-cube-look-up-tables-and-other-random-words/example.png)

### Color Cubes and Lookup Tables

It has been over 10 years since I released PowerUp, an iOS app that converts and image to a pixelated version using a color pallet. Behind that was [Core Image](https://developer.apple.com/documentation/coreimage), iOS' image processing framework. The pixelation part was easy but the color pallet part was tough. How on earth would you constrain a modern image using a certain set of colors?

Well, that's what a [Lookup Table (LUT)](https://en.wikipedia.org/wiki/Lookup_table) is for. In the simplest term, let's say you have an image that is one color, solid red. If it was a RGBA image, the pixel would be comprised of values 255 (red), 0 (green), 0 (blue), 255 (alpha). Let's say we wanted to change that to green, we'd pass a look up table that would instruct the GPU to replace all instances of (255, 0, 0, 255) with (0, 255, 0, 255). This is how they got [Ryu into all those outfits](https://images-wixmp-ed30a86b8c4ca887773594c2.wixmp.com/f/04fdc7f8-00fa-4b43-aa4a-6cc308cee66f/d4befue-51e0b74c-4b95-47d9-bea0-85e9b025ea95.png?token=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ1cm46YXBwOjdlMGQxODg5ODIyNjQzNzNhNWYwZDQxNWVhMGQyNmUwIiwiaXNzIjoidXJuOmFwcDo3ZTBkMTg4OTgyMjY0MzczYTVmMGQ0MTVlYTBkMjZlMCIsIm9iaiI6W1t7InBhdGgiOiJcL2ZcLzA0ZmRjN2Y4LTAwZmEtNGI0My1hYTRhLTZjYzMwOGNlZTY2ZlwvZDRiZWZ1ZS01MWUwYjc0Yy00Yjk1LTQ3ZDktYmVhMC04NWU5YjAyNWVhOTUucG5nIn1dXSwiYXVkIjpbInVybjpzZXJ2aWNlOmZpbGUuZG93bmxvYWQiXX0.gqZOvR_jh5PuYER8NocK3V_Pdl4_5uSHCsoLAaqIyeI).

 Apple calls this a “Color Cube” for no good reason and you can utilize it in Core Image with [CIColorCube](https://developer.apple.com/documentation/coreimage/cicolorcube). With super clear and instructive documentation, you feed this filter with the lookup table in the form of data. Data that is, “should be an array of texel values in 32-bit floating-point RGBA linear premultiplied format.”. EASY!

 Back in 2012, Apple released a WWDC video outlining this with the example of a [chroma keying](https://en.wikipedia.org/wiki/Chroma_key). This was also before Swift so you could fudge a lot of things by throwing pixel component values into a block of memory allocated to be a `char` pointer of a certain size. That's just what I did when I wrote [InstaCube](https://github.com/jacobvanorder/InstaCube). The idea was that you'd apply a filter to the key image and then use that as a visual representation of the lookup table. Effectively, each pixel in the image you gave it was a set of values that Core Image would know the corresponding values to and replace it with.

Fast-forward to 2023. Remember, I'm writing an [Instagram clone](https://jacobvanorder.github.io/mastodon-auth/) and Instagram has the ability to apply an uniform filter over an image. Sounds like a perfect case for our old friend Color Cube. But, that Objective-C and static library. That's a “no go” with SwiftUI so let's rewrite it. 

### Converting 10+ Year Old Code

The main sticking point was how to pull the data out of the image and convert it into data in the format that Core Image is expecting but what does that look like? After all, in Objective-C, [just throw](https://github.com/jacobvanorder/InstaCube/blob/master/InstaCubeExample/InstaCube/InstaCubeGenerator.m#L93) whatever it is into whatever you made. In Swift, that's a no-no. You *must* know what you're dealing with. In this case, I kept running into issues because the description above says “32-bit floating-point RGBA linear premultiplied format” so I assumed it needed to be an array of `Float`. 

Getting the pixel data could be done a number of ways:

* Converting the image to a `cgImage` and then using [`CGDataProvider`](https://developer.apple.com/documentation/coregraphics/cgimage/1455260-dataprovider)
* Drawing the image into a `CIContext` using a [rendering method](https://developer.apple.com/documentation/coreimage/cicontext)
* Creating a `CGContext`, drawing into it, and getting the [data](https://developer.apple.com/documentation/coregraphics/cgcontext/1455517-data#)

The first one does not work but the second two do. But after that, you're given the super clear and not at all opaque `UnsafeMutableRawPointer`. It's funny, even now I turn to [Ray Wenderlich](https://www.kodeco.com/7181017-unsafe-swift-using-pointers-and-interacting-with-c) to explain things better than the docs can. Essentially, I needed to convert that to a `UnsafeMutablePointer` which would be typed. But which type? Remember, I thought it was a `Float` but, after much stabbing into the dark, it turned out to be `UInt8` since the values are between 0 and 255. I was able to pull those values out as `UnsafeMutablePointer` has subscripting and it was as easy as calculating the image's `width * height * 4` with `4` representing R,G,B,A and, for each component pulling it out directly and storing into an array of `UInt8`. 

But how to convert that to `Data`? We are dealing with value types and it seems totally not able but it's a matter of throwing a `&` infront of the collection and using `NSData(bytes: length:)`. 

Whew. 

The result is [here](https://github.com/jacobvanorder/InstaCube_Swift) but there was one more sticking point. No matter what I tried, the result wasn't right. It was slightly too dark. It turns out that Apple released [`CIColorCubeWithColorSpace`](https://developer.apple.com/documentation/coreimage/cicolorcubewithcolorspace) which takes into consideration the slight differences in how images are processed. It's just a matter of making sure that the images, context, and color cube are all using the same color space and it works!
