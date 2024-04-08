---
layout: post
title: Presenting 3D Assets on Vision Pro
date: 2024-03-23 11:36 -0500
---

# Vision Pro!

Let's not get into how $3,500 could be better spent, if this is [really the best time for the release](https://en.wikipedia.org/wiki/Microsoft_Tablet_PC) of this hardware, or if iPadOS was the best choice of a platform to base "spatial computing" on. 

It is what it is. 

I truly believe that a form of what the Vision Pro **is** _will be_ integral to computing in the future. I don't think it's as good as the first iPhone in terms of hitting the target right at the start but I hope it's not like the iPad where a promising beginning is hampered by being tied to an operating system that limits it. 

## Presenting 3D Models

I don't think that, unlike the phone, the compelling mode of the Vision Pro is looking at an endless scrollview of content. Instead, being able to see a 3D asset in stereo 3D gives the most bang for the buck. Watching a movie on a place on screen the size of a theater is cool but watching truly immersive material wherever you are is that much more special and worth the tradeoffs of having a hunk of metal and glass strapped to your face. 

### Different Methods of Presenting a 3D Asset

As I [posted before](https://jacobvanorder.github.io/a-glimpse-into-the-future/), there are ways to generate 3D assets using your phone. As a brief update, Apple has released this functionality now [completely on your phone](https://developer.apple.com/videos/play/wwdc2023/10191/) and the results are spectacular. You can generate a 3D model using your iPhone but, grumble, grumble, not on your $3,500 Vision Pro. 

Unlike on the phone, though, VisionOS provides very easy ways to present the 3D content to the user whether embedded within the user interface or within a more freeform manner. In this post, we'll be touching on the more simpler form of presenting 3D content. The more complicated form, `RealityView` could fill a series of blog posts which I'll be tackling.

The sample code for these examples is [here](https://github.com/jacobvanorder/VisionProPlacement). 

#### Model3D

`Model3D` is a [SwiftUI view](https://developer.apple.com/documentation/realitykit/model3d/) that, in the words of the documentation: "asynchronously loads and displays a 3D model". This, though, undersells it's capability. It can do it from a local file OR a URL. Because both of these methods can be time consuming, it is done in a way similar to the [`AsyncImage`](https://developer.apple.com/documentation/swiftui/asyncimage) view does for images loaded from the network. 

This means that you have the view itself but then, when the 3D asset is loaded, you are presented with a [`ResolvedModel3D`](https://developer.apple.com/documentation/realitykit/resolvedmodel3d) that you can then alter.

##### Animation

Given we have this 3D asset in our space, we can then animate the content just like we would a normal SwiftUI view but, again, this will need to be done to the `ResolvedModel3D` content. The traditional way of a continuous animation would be where you add a `@State` property that keeps track of if the content has appeared and uses that to use as the basis for the before and after values for the animation. Then it is a matter of using the new `rotation3DEffect` on the resolved content. Alternatively, you can use the new `PhaseAnimation` and not have a need for the `@State` property. 

For either way you go, beware that the layout frame might not be what you expect. Alternatively, because the layout is based on the width and the height of the model _but not the depth_, when you rotate along the y-axis, the depth will now become the width and you layout might look wrong. You can utilize the new `GeometryReader3D` in order to gather the height, width, and now depth of the view and adjust accordingly. 

##### Gestures

For both examples, we'll be modifying the views with `.gesture` but whichever gesture we choose, we need to tell the view that these will apply not to the view but to the entity contained within via the `.targetedToAnyEntity()` modifier on the gesture. You can also specify which entity you want to attach the gesture to by using `.targetedToEntity(entity: Entity)` or `.targetedToEntity(where: QueryPredicate<Entity>)`. The `.onChanged` and `.onEnded` modifier will now have 3D-specific types passed in.

###### Drag Gestures

We can use a traditional `DragGesture` in order to rotate the content using the `rotation3DEffect` we used for the animation. In the [example](https://github.com/jacobvanorder/Presenting3DContent/blob/main/Presenting3DContent/Examples/Model3DDragGestureView.swift), we have a `startSpinValue` and a `spinValue` that we'll be keeping track of. The difference between the two is that `startSpinValue` is sort of the baseline value that we keep track of while the drag gesture is happening. We get the delta of the drag by calculating the difference between the start and current position and applying that _plus_ the `startSpinValue` to set the `spinValue`. If we did not have the `startSpinValue`, if we were to rotate the entity for a second time, it would begin rotating from 0.0 and not from the previous value we rotated to. 

###### Rotate Gesture 3D

Because this is the Vision Pro, you can rotate by pinching both of your hands and acting like you're turning a wheel in space in order to rotate an item. This will allow you to save your drag gesture for when you want to move your item but still reserve the ability to rotate it. The [code](https://github.com/jacobvanorder/Presenting3DContent/blob/main/Presenting3DContent/Examples/Model3DRotateGestureView.swift) for this example is different because we don't store a value of amount we are spinning the entity but we do have an optional value that is meant to be the baseline value of the rotation that has happened. Additionally, we don't use the `rotation3DEffect` and instead change the entity's `transform` value by _multiplying_ the baseline value times the gesture's rotation value.  I added `Model3DDragGestureAltView` in order to show how you might do this way of rotating the item using the drag gesture. 

##### Gotchas

Because you have a container of the `Model3D` and then the actual content of the `ResolvedModel3D`, you can get into a situation where the layout frame of the container might not be what you expect it to be based on the actual content.

###### Sizing

Just like `AsyncImage`, the view doesn't know the resulting content size. Usually, it "just works" but if you start animating or altering the resolved 3D content, be aware that you're not dealing with both width and height but also depth.

For instance, because it defaults to placing the content where the back is placed against the front of the view you are placing it in, perspective and the depth of the model might hide the other content in the `VStack` or `HStack` so be mindful. 

![Hidden text under a 3D Model](/assets/images/2024-03-23-presenting-3d-assets-on-vision-pro/hidden-text.png)

###### View Modifiers

Because these are all extensions on `View` that throw a view modifier into the great next responder chain that is `@environment`, view modifiers such as `blur(radius:)` or `blendMode(_:)` don't work but `.opacity(_:)` _does_ (grumble, grumble, grumble). 


  