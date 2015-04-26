---
layout: post
title: "Windows Store Apps, WASAPI, and Audio Code in C#"
tags: [Audio, Windows]
---
In my recent talk at //Build ([Sequencers, Synthesizers, and Software, Oh My!](https://channel9.msdn.com/Events/Build/2014/3-548)), 
I showed how WASAPI could be used to create low-latency audio apps
for Windows Store. You can find the Beat Builder sample code [published on GitHub](https://github.com/jolson88/BeatBuilder) 
under the MIT License. A question that has come up though is how you could have all your sound synthesis code
in C#. So let's dig into how to do that in this blog post.

*NOTE: This post doesn't dive into using 3rd party libraries like NAudio, PortAudio, Jack, etc. 
Perhaps in a future post we will get into some of these :).*

Just for sake of consistency with the Beat Builder sample code, I'm going to keep the core of my
audio renderer (the code that hooks into WASAPI) in C++. But what I want to do is expose a way for 
other languages (yes, even JavaScript) to be able to generate audio data that is going to be 
rendered by this core code. Anybody who has written audio generation code will know (whether 
WASAPI-based, VST-based, etc.), it boils down very basically to one function:

{% highlight c++ %}
void Process(float* bufferToFill, int bufferLength);
{% endhighlight %}

So here's our first challenge. WinRT doesn't have the concept of a pointer. WinRT does have arrays, 
but there’s one issue we have to think about: memory copying. Because of how tight the audio rendering 
loop is (and how often it’s called), we need to be very conscious of the number of times our buffer is 
copied. We want to avoid having too many excessive temporary objects/copies to avoid churning the 
generational garbage collector in the CLR. The last thing we want to do is trigger extra collection 
runs and introduce potential audio glitches into our synthesis engine.

Ideally, we will never copy our buffer. Doing buffer copies when you cross a language boundary in WinRT 
would be A Bad Thing. So arrays are out. But without Pointers, how do we represent a section of memory 
to be populated with data? The [Windows.Storage.Streams.Buffer class](http://msdn.microsoft.com/en-us/library/windows/apps/windows.storage.streams.buffer) (and the IBuffer interface). Upside: .NET has some good helpers to make working with Buffers easier by interop’ing with Arrays and such. Downside: Each one of these mechanisms copy the underlying buffer first! 
And looking at the class, there’s no methods or properties that allow us to actually get to the underlying 
data… So how do we actually do this?

Windows.Storage.Streams.Buffer not only implements the IBuffer WinRT interface, it also implements a COM 
interface named IBufferByteAccess. By 
querying an IBuffer interface for the IBufferByteAccess COM interface, we can then get access directly 
to the buffer’s backing byte* data through IBufferByteAccess’s Buffer property. To do this in C#, we need 
to define this COM interface using the standard COM interop mechanisms in the CLR:

```csharp
{% highlight c# linenos %}
[ComImport]
[Guid("905a0fef-bc53-11df-8c49-001e4fc686da")]
[InterfaceType(ComInterfaceType.InterfaceIsIUnknown)]
internal interface IBufferByteAccess
{
	unsafe byte* Buffer
	{
		get;
	}
}
{% endhighlight %}
```

In an unsafe method, we can now use this to get the underlying byte*, cast it to a float* for our audio code, 
and start filling the buffer from C# (that was passed to us across the WinRT boundary from C++ using the Buffer 
runtime class and IBuffer interface).  This will of course require you to compile your project with the “Allow 
Unsafe Code” option set since you will need unsafe code to write pointers in C#.

{% highlight c# %}
public unsafe void FillSamples(IBuffer bufferToFill)
{
	IBufferByteAccess byteAccess = (IBufferByteAccess)bufferToFill;
	float* dataPtr = (float*)byteAccess.Buffer;
     
	// Now we can have dataPtr[0]...dataPtr[n] to generate our samples
	// without any extra copies
}
{% endhighlight %}

Rather than requiring unsafe to litter my code in different places, I’ve created a helper class (FloatBuffer) in 
the BeatBuilder sample code on GitHub to make it easier to work with:

{% highlight c# linenos %}
    using System;
    using System.Collections.Generic;
    using System.Runtime.InteropServices;
    using System.Text;
    using Windows.Storage.Streams;
     
    namespace BeatBuilder.Audio
    {
        public unsafe class FloatBuffer
        {
            private IBuffer m_buffer;
            private float* m_dataPtr;
     
            public uint Length
            {
                get { return m_buffer.Capacity / sizeof(float); }
            }
     
            public float this[int key]
            {
                get
                {
                    if (key > this.Length)
                        throw new IndexOutOfRangeException();
     
                    return m_dataPtr[key];
                }
                set
                {
                    if (key > this.Length)
                        throw new IndexOutOfRangeException();
     
                    m_dataPtr[key] = value;
                }
            }
     
            public FloatBuffer(IBuffer buffer)
            {
                m_buffer = buffer;
                m_dataPtr = (float*)((IBufferByteAccess)buffer).Buffer;
            }
        }
    }
{% endhighlight %}

Using this helper class, I can create a simple lookup-table-based Oscillator class in C# that generates a pure 
Sin wave that can plug in to my C++/CX AudioRenderer class. That way, only the base WASAPI hooks are in C++ and 
my audio generation code can remain in C# if I want it that way.

{% highlight c# linenos %}
    using System;
    using System.Collections.Generic;
    using System.Text;
    using Windows.Storage.Streams;
     
    namespace BeatBuilder.Audio
    {
        public class Oscillator : ISoundSource
        {
            private static int LOOKUP_SAMPLE_COUNT = 2048;
     
            private bool m_isOn = false;
            private double m_frequency = 440;
            private double m_phaseDelta;
            private double m_phaseAccumulator;
            private float[] m_sampleLookup;
            private WaveFormat m_waveFormat;
     
            public Oscillator()
            {
                m_sampleLookup = new float[LOOKUP_SAMPLE_COUNT];
     
                // Build just a sin-wave lookup table for now
                for(int i = 0; i < m_sampleLookup.Length; i++)
                {
                    m_sampleLookup[i] = (float)Math.Sin((2 * Math.PI) * (i / (double)LOOKUP_SAMPLE_COUNT));
                }
            }
     
            private WaveFormat WaveFormat
            {
                set
                {
                    m_waveFormat = value;
                    Recalibrate();
                }
            }
     
            public void TurnOn()
            {
                m_isOn = true;
            }
     
            public void TurnOff()
            {
                m_isOn = false;
            }
     
            public void FillNextSamples(IBuffer bufferToFill, int frameCount, int channels, int sampleRate)
            {
                if (m_isOn)
                {
                    var floatBuffer = new FloatBuffer(bufferToFill);
     
                    // The lookup table is just Mono, so need to output appropriate channels
                    for (int i = 0; i < floatBuffer.Length; i=i+channels)
                    {
                        for (int j = 0; j < channels; j++)
                        {
                            floatBuffer[i+j] = m_sampleLookup[(int)m_phaseAccumulator];
                        }
     
                        m_phaseAccumulator = (m_phaseAccumulator + m_phaseDelta) % LOOKUP_SAMPLE_COUNT;
                    }
                }
            }
     
            public void SetWaveFormat(WaveFormat format)
            {
                this.WaveFormat = format;
            }
     
            private void Recalibrate()
            {
                // The wave format of oscillator frequency has changed, so recalibrate oscillator
                m_phaseDelta = (LOOKUP_SAMPLE_COUNT * m_frequency) / m_waveFormat.SamplesPerSecond;
            }
        }
    }
{% endhighlight %}

As mentioned above, all of this is now reflected in the [Beat Builder project on GitHub](https://github.com/jolson88/BeatBuilder). 
Just check out Oscillator.cs and BufferHelpers.cs (and AudioRenderer.cpp if you want to see the hook into 
WASAPI and how I call out to sound generators, a.k.a. classes that implement ISoundSource that the renderer 
is listening to). 
