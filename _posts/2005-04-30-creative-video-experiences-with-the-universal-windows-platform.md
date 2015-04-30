---
title: Creative Video Experiences with the Universal Windows Platform
tags: [Programming,Windows]
---
I'm excited that //build is finally here! It means that I can finally come out and share what I've been up to over the last year. My partner-in-crime Bala Sivakumar and I presented a talk at //build called ["A studio in the palm of your hand: Developing audio and video creation apps for Windows 10"](http://channel9.msdn.com/events/Build/2015/3-634) that should be available on Channel 9 in the coming days.

As a creative person, I love that I have devices always within my reach. It means that whenever and wherever inspiration hits, I can capture it easily before the muse leaves me. But for the best experience, I need great creative apps that I can use to capture that inspiration. But building these apps on Windows previously has taken a lot of effort, and even more domain knowledge when working with our APIs. In Windows 10, we take care of many of the moving pieces so app developers can focus on the features that will keep their users coming back for more and more.

### Creatively working with video
On Windows 8.1, working with video in creative ways usually meant dipping your toes down into the depths of Media Foundation. While Media Foundation is a very powerful and very flexible API, that comes along with a lot of extra responsibilities that app developers have to take care of themselves. This in turn means developers spend more time focusing on the pipeline and infrastructure work instead of features that add unique value to their app. 

Step 1. In Windows Phone 8.1 we introduced a new API tailored at working with video and images in creative ways: Media Composition.

> **Media Composition** is an API in the Windows.Media.Editing namespace that is part of the Universal Windows Platform. Media Composition provides a lot of foundational technology when working with videos and video editing that allows app developers to focus on the features that set their app apart. Historically, simple operations like trimming videos, appending videos together, applying effects to video, etc. used to take many thousands of lines of code using Microsoft Media Foundation. Now, we take of a lot of the plumbing for you so you can perform all these basic scenarios with < 10 lines of code.

With this start, there was an easy first decision for Windows 10: incorporate Media Composition into the Universal Windows Platform. With Windows 10 developers can now use Media Composition across the wide variety of device families that support the Universal Windows Platform, from Phones, to Desktops, to Xbox, etc. 

We noticed that there were still basic creative scenarios that required app developers to drop down into Media Foundation, off the end of the complexity cliff. Many creative video experiences really need creative filters and effects to keep users engaged and to unlock their creativity. Also, whether you are developing a news broadcast app, a basic video editing, or an app that generates memoir videos, developers are doing a lot of work now to overlay one video or photo on top of another video or photo. 

#### A New Effect Model
So do you really fall off a complexity cliff when implementing effects with Media Foundation? The official MSDN sample for implementing a Grayscale effect in a Windows store app using a Media Foundation Transform... around 2000 lines of code. That's a lot of code! Having to spend so much development effort focused on this plumbing code is not a good recipe for success for app developers. 

To help app developers, we introduced a brand new video and audio effect model when working with media APIs on the Universal Windows Platform. All it takes is to implement either the IBasicVideoEffect or IBasicAudioEffect interfaces. 

> **IBasicVideoEffect** and **IBasicAudioEffect** are new Universal Windows Platform interfaces that developers can implement to create their own effects that can be used with many different media APIs: Media Element, Media Capture, Media Composition, and Media Transcoder. These new interfaces significantly reduce the need to dip down and create custom Media Foundation Transforms.

The heart of these interfaces are in one method: ProcessFrame(). The media pipeline will give you the incoming frame, you do whatever you want to do to process it, and you give the media pipeline back the resulting output frame. IPropertySet is also used as a mechanism to configure effects. This has the added benefit of being a mechanism that can be used to communicate between different types of effects!

One of the things I enjoy most about these interfaces is that the two fundamental ways to get the video frame are key interop types that are used with other graphics processing libraries. Namely, SoftwareBitmap for CPU-based video frames (allowing the use of Wic, Lumia Imaging SDK, etc.), and IDirect3DSurface for GPU-based video frames (allowing the use of Win2d, Direct2d, Direct3d, etc.). 

For example, to implement a very simple desaturation effect to make a video completely desaturated, we can do the bulk of the graphics code in just a couple of lines of C# code (all graphics-accelerated of course!):

{% highlight c# linenos %}
    
    public void ProcessFrame(ProcessVideoFrameContext context)
    {
        using (var inputBitmap = CanvasBitmap.CreateFromDirect3D11Surface(                             
                                      _canvasDevice,
                                      context.InputFrame.Direct3DSurface))
        using (var renderTarget = CanvasRenderTarget.CreateFromDirect3D11Surface(
                                      _canvasDevice,
                                      context.OutputFrame.Direct3DSurface))
        using (var ds = renderTarget.CreateDrawingSession())
        {
            var saturation = new SaturationEffect()
            {
                Source = inputBitmap,
                Saturation = 0.0f
            };
            ds.DrawImage(saturation);
        }
    }

    public void SetEncodingProperties(VideoEncodingProperties encodingProperties, IDirect3DDevice device)
    {
        _canvasDevice = CanvasDevice.CreateFromDirect3D11Device(device, CanvasDebugLevel.Error);
    }
    
{% endhighlight %}

#### Overlays
Another common scenario when working with video is to add overlays. By supporting overlays, we can very easily overlay video on video, photo on video, photo on photo, etc. We can even implement other features like transitions between videos. 

With Windows 10, we didn't want to limit it to just a single overlay either. So we give you full control over the number of overlays you can add, how they are grouped, and even how they are rendered. If we want to overlay four videos painted in ellipses over another video, you can do (just for starters): 

![Several videos overlayed using ellipses on a base video]({{ site.baseurl }}/images/posts/simple-overlays.png)

Out of the box, MediaComposition uses a very basic video compositor that allows you to specify the location of the overlay and an opacity. MediaComposition then composes each frame using simple alpha-blending. Where the power of overlays really starts to shine is when you start implementing your own custom video compositors. 

> A **custom video compositor** is a class that implements the IVideoCompositor interface and is passed to an overlay layer. Via the CompositeFrame method, the media pipeline gives the custom video compositor all the overlays that need to be composed into the final composite frame. The compositor gives the resulting frame back to the media pipeline in which it is used in the rest of the pipeline.

So let's say we want to create a simple custom video compositor. Similar to effects, implementing a custom video compositor is all about implementing a single interface: IVideoCompositor. To see how simple the CompositeFrame method is to implement, let's see how we would overlay our videos as a brush using simple ellipse shapes.

{% highlight c# linenos %}
    
    public void CompositeFrame(CompositeVideoFrameContext context)
    {
        var outputSurface = context.OutputFrame.Direct3DSurface;
        using (var renderTarget = CanvasRenderTarget.CreateFromDirect3D11Surface(_canvasDevice, outputSurface))
        using (var drawSession = renderTarget.CreateDrawingSession())
        {
            foreach(var overlaySurface in context.SurfacesToOverlay)
            {
                var overlay = context.GetOverlayForSurface(overlaySurface);
                var width = (float)overlay.Position.Width;
                var height = (float)overlay.Position.Height;
                using (var overlayBitmap = CanvasBitmap.CreateFromDirect3D11Surface(_canvasDevice, overlaySurface))
                using (var videoBrush = new CanvasImageBrush(_canvasDevice, overlayBitmap))
                {
                    var scale = width / overlay.Clip.GetVideoEncodingProperties().Width;
                    videoBrush.Transform = CreateScalingAndTransformMatrix(scale, (float)overlay.Position.X, (float)overlay.Position.Y);
                    drawSession.FillEllipse(new Vector2((float)overlay.Position.X + width / 2, (float)overlay.Position.Y + height / 2), width / 2, height / 2, videoBrush);
                }
            }
    
            drawSession.DrawText("Happy\nBirthday!", new Vector2(_backgroundProperties.Width / 1.5f, 100), Windows.UI.Colors.CornflowerBlue, new CanvasTextFormat()
            {
                FontSize = (float)_backgroundProperties.Width / 13,
                FontWeight = new FontWeight() { Weight = 999 },
                HorizontalAlignment = CanvasHorizontalAlignment.Center,
                VerticalAlignment = CanvasVerticalAlignment.Center
            });
        }
    }

{% endhighlight %}

That's not all you can do with custom video compositors though. You can easily animate different overlays together, implement your own custom transitions, and much much more. 

This is just the start of the features we are building into the creative media experience as part of the Universal Windows Platform. We also have new APIs in the creative audio and MIDI spaces that we discussed at //build and that I will no doubt blog about in the future. We will continue to build on top of this new foundation with each successive release so there are a lot of fun things to come :).

### If you have any feedback...
We'd love to hear from you if you have any feedback using these APIs. We want to continue to provide a great experience when working with video, whether it's on a simple video editor, or if it's doing out-of-the-box 3d projection work working with different forms of videos displayed on different types of objects. Or, anything crazy we haven't conceived of :).

All of the work we've done is possible because of the great feedback we've heard from app developers. We would love for this to continue as we are here to help you build the best apps possible!
