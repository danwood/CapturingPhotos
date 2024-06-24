# Writing an iPhone Camera App with SwiftUI

> [!NOTE]
> Based on [this Apple Sample Code](https://developer.apple.com/tutorials/sample-apps/capturingphotos-camerapreview). This README is based on article originally published at [https://www.linkedin.com/pulse/writing-iphone-camera-app-swiftui-dan-wood-xru4c](https://www.linkedin.com/pulse/writing-iphone-camera-app-swiftui-dan-wood-xru4c).

Over the last several years, I've worked on several iPhone projects that make use of the built-in camera. Developers who have written similar apps have probably started with some Apple sample code, such as [AVCam: Building a camera app](https://developer.apple.com/documentation/avfoundation/capture_setup/avcam_building_a_camera_app?trk=article-ssr-frontend-pulse_little-text-block).

This sample code has been updated through the years as new frameworks became available. The version currently online, updated just after WWDC24, ostensibly requires iOS 18 (beta) to run, though I have found that it will also deploy to iOS 17 with only a minor change to project settings.

As I've moved almost all development to SwiftUI in favor of the older UIKit and AppKit frameworks, the lack of a "native" SwiftUI approach has bugged me. Even the latest update to this sample code, in its README, implies that the only way to create a camera preview is to build a `UIViewRepresentable`, wrapping the `UIView` that holds an `AVCaptureVideoPreviewLayer`.

But it turns out there is a way to build a camera app with a native SwiftUI view rather than a wrapped `UIView`! A couple of weeks ago, I stumbled upon [Capturing and Displaying Photos: Previewing the Camera Output](https://developer.apple.com/tutorials/sample-apps/capturingphotos-camerapreview?trk=article-ssr-frontend-pulse_little-text-block). According to the [WayBack Machine](https://wayback-api.archive.org/?trk=article-ssr-frontend-pulse_little-text-block), this code was first published in May 2022! And yet, none of the developers I mentioned this to last week at the [One More Thing](http://omt-conf.com/?trk=article-ssr-frontend-pulse_little-text-block) conference were aware of this code and the technique it espouses.

The sample code is fairly straightforward, but I did find some room for improvement and simplification. **Never assume that Apple's sample code is 100% perfect or uses all best development practices!** So let's see how Apple's code works, then analyze it.

Apple Sample Code Architecture
-----------

![3 main objects: Camera, Data Model, Viewfinder View](images-for-readme/orig-arch.png]

# Architecture for Apple's Sample Code for Previewing the Camera Output

Stripping away the code for saving the photos, the capturing architecture that remains is quite simple:

*   The `Camera` object sets up all of the AVFoundation inputs and outputs. It implements the `AVCaptureVideoDataOutputSampleBufferDelegate` protocol to process each frame that comes in from the camera. With each frame, it creates a `CIImage` from the sample buffer's pixel buffer. Each of these images is added to an `AsyncStream<CIImage>`.
*   The `DataModel`'s job — along with saving captured photos when the shutter is pressed — is to wait for each image in the above stream (using `for await`), convert it to a SwiftUI `Image` object, and update the value of its `viewfinderImage`, a `@Published` property.
*   The `ViewFinderView` will repaint each time the `viewfinderImage` is updated, thanks to SwiftUI's magic.

That's it! By keeping all of the AVFoundation code encapsulated in the `Camera` object, and making use of async streams, the `ViewfinderView` works while avoiding the convoluted `UIViewRepresentable` technique.

Performance
-----------

I worried when I saw this code that it might not be performant enough to use in shipping code. I did some quick tests using an iPhone SE — not the most powerful of Apple's current lineup! — and found that it didn't break a sweat. The performance of this sample code was almost as good as the "AVCam" camera sample code which uses the `UIViewRepresentable` technique. The "Energy Impact" meter in the pure SwiftUI code was quite comfortable in the "low" zone. The `UIViewRepresentable` code's energy impact was often "none," so if you are trying to squeeze out *every last bit of battery life** from your code, especially if you already have working code, you can skip the rest of this article and stick with your legacy code. (In my current project, I was creating a `CIImage` for each frame anyhow, in order to run through some `CoreML` and `Vision` processing, so it was worth it for me to change and simplify my codebase.)

![meter showing low energy impact](images-for-readme/low-energy.png]

Room for Improvement
--------------------

I found several opportunities to tighten up the sample code. If you wish to use this `UIView`-less technique, you may want to consider these for your code.

*   Don't create a new `CIContext` with every frame. Apple's code converts each `CIImage` instance into a SwiftUI `Image` by allocating a new `CIContext`, then building a `CGImage`, then building a SwiftUI `Image`. This is happening with every frame from the camera! Instead, create a *single** CIContext and use that for *all* of the conversions.
*   For the `CIContext`, be sure to allocate it with the right parameters. In a [WWDC 2020 talk](https://developer.apple.com/videos/play/wwdc2020/10008/?trk=article-ssr-frontend-pulse_little-text-block), the presenter said "Most important is to tell the context not to cache intermediates. With video, every frame is different than the one before it, so disabling this cache will lower memory usage." So be sure to set `.cacheIntermediates` to false.
*   The `ViewfinderView` has a `@Binding` to the image that is passed in as a projected value with the `$` prefix. The image, however, is *not modified in the view*. This should be a simple `let` parameter directly passed in!
*   In my code, I moved up the ownership of the `Camera` object, out of the `DataModel`, and into the parent view. I haven't yet tried mocking a camera for testing, but if I do, it will be easier to use dependency injection.
*   Depending on the direction you go with your code, you may want to change the `DataModel` from an `ObservableObject` subclass to an `@Observable`. [This might improve performance.](https://www.avanderlee.com/swiftui/observable-macro-performance-increase-observableobject/?trk=article-ssr-frontend-pulse_little-text-block)
*   Even better, you may be able to remove the `ViewModel` class altogether. If all it is doing is serving as a bridge between the async stream and the current frame, you could just set up a task in a SwiftUI View which awaits each new frame and updates a `@State` variable holding the current frame (`viewfinderImage`) to trigger a refresh.
*   There isn't any disadvantage to creating a SwiftUI `Image` object as a property to be updated. Still, if your code is like mine and needs to process each bitmap frame, it may be easier to change your stream to vend lower-level image objects, closer to the source data, such as `CVPixelBuffer`, `CGImage`, or `CIImage`. (Any of these are suitable input to a Vision image request for CoreML processing or text recognition.) It's easy to construct a SwiftUI `Image` as needed for display, but since that is an opaque type — only usable by SwiftUI — you can't go in the other direction.
*   The sample code doesn't specify the video settings for the video output in your camera capture session, and fortunately, it seems to default to `kCVPixelFormatType_420YpCbCr8BiPlanarVideoRange`. It's important to use a [performant video pixel format](https://developer.apple.com/documentation/technotes/tn3121-selecting-a-pixel-format-for-an-avcapturevideodataoutput?trk=article-ssr-frontend-pulse_little-text-block). I'd rather not leave that to chance in my code so I'd recommend setting the format explicitly.

New Architecture
----------------

![3 main objects: Camera, Content View, Viewfinder View](images-for-readme/new-arch.png]

Possible new architecture for previewing the camera with a SwiftUI view

I am more comfortable with the modified architecture; it's a bit more straightforward than Apple's sample code. I hope you agree, and I hope you can make use of these insights in your projects!

