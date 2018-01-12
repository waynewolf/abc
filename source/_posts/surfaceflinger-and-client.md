---
title: surfaceflinger and client
date: 2014-06-22 18:43:49
tags:
- android
---

## Introduction
SurfaceFlinger is the composition and display manager of android, it mainly plays two roles in the system:

* Surface composition
* Buffer allocation on behalf of clients, interacts with clients through Binder interface


This article is based on KitKat 4.4.2.

<!--more-->
## How client draws UI
ISurfaceComposer is the interface to talk to SurfaceFlinger, there're two ways to get ISurfaceComposer interface in Client:

	sp<ISurfaceComposer> composer;
	composer = ComposerService::getComposerService();

or

	getService("SurfaceFlinger", &composer);

ISurfaceComposerClient is the interface to create Surface. To get ISurfaceComposerClient interface, use the ISurfaceComposer instance returned above:

	sp<ISurfaceComposerClient> composerClient = composer->createConnection();
	
The above two methods can be combined in single step:

	sp<SurfaceComposerClient> composerClient = new SurfaceComposerClient;

To operate on Surface, you have to get SurfaceControl first, simply call createSurface on the SurfaceComposerClient instance:

	sp<SurfaceControl> surfaceControl = composerClient->createSurface();

And call getSurface on SurfaceControl instance, you can finally get Surface:

	sp<Surface> surface = surfaceControl->getSurface();

Get buffer to draw:

	int Surface::lock(ANativeWindow_Buffer* outBuffer, ARect* inOutDirtyBounds);

Draw on surface with whatever method you want.

Complete drawing by calling:

	int Surface::unlockAndPost();

If some operations need atomic transaction, just wrap them inside:

	SurfaceComposerClient::openGlobalTransaction();
	...
	SurfaceComposerClient::closeGlobalTransaction();

## BufferQueue and Consumer/Provider
BufferQueue plays a central role in GraphicBuffer exchange process, two roles which are consumer and provider are designed to distinguish different entity and intension.
The producer of a BufferQueue can

    queueBuffer
    dequeueBuffer

The consumer of a BufferQueue can

    acquireBuffer
    releaseBuffer

Back to BufferQueue, usually client application create Surface, it's provider end of a BufferQueue, SurfaceFlinger acts as consumer of the Surface provided by client. However, SurfaceFlinger also acts as provider as regard to FrameBufferSurface, which is consumed by display.

These APIs support android native sync to manage concurrency. Android native sync is introduced in 4.x.x. It's a framework used by native code, while at the same time, graphic drivers haver to implement correspondance interface. Android sync is out of the scope of this article.

## Drawing a frame

### General flow

<pre>
handleMessageRefresh
     |—> preComposition
     |—> rebuildLayerStacks
     |—> setUpHWComposer
          |—> HWComposer::createWorkList <== hwc structures are allocated
          |—> Layer::setGeometry()
          |—  set per frame data
          |—  HWComposer::prepare
               |—> hwc prepare
     |—> doComposition
          |---- skip composition on external display if condition meets
          |—> doDisplayComposition
          |    |—> doComposeSurfaces
          |     |—> DisplayDevice::swapBuffers
          |          |—> eglSwapBuffers
          |          |—> FramebufferSurface::advanceFrame
          |—> DisplayDevice::flip(…)     <== just update statistics count
          |--> Call DisplayDevice::compositionComplete(), notify each display
               |--> DisplaySurface::compositionComplete()
                    |--> FramebufferSurface::compositionComplete()
                         |--> HWComposer::fbCompositionComplete()
                              |--> NoOP if HWC >= 1.1
                              |--> used only in framebuffer device case.
          |—> postFrameBuffer
               |—> HWComposer::commit
                    |—> hwc set
                    |—> update retireFenceFd of hwc_display_contents_1
               |—> DisplayDevice::onSwapBuffersCompleted
                    |—> FramebufferSurface::onFrameComitted
               |—> Layer::onLayerDisplayed
               |—   update some statistics
     |—> postComposition 
</pre>

### doComposeSurfaces
handleMessageRefresh -> doComposition -> doDisplayComposition -> doComposeSurfaces

* Preparation work: 
    * If GLES and hwc compositing, clear frame buffer target first 
    * If GLES only, drawWarmHole first
* Render layers to framebuffer
    * For all layers if using hwc
        * do nothing if HWC_OVERLAY layer, display hardware will blend the layer
        * render with opengl if HWC_FRAMEBUFFER layer, call layer->draw()
        * set the layer's acquireFence
    * For all layers if no hwc
        * just render with OpenGL, call layer->draw()
* Now all the GLES layers are drawn on frame buffer target, waiting to swapBuffers

HWC related topic will be written in another article.

### How FrameBufferSurface and eglSwapBuffers interact
In SurfaceFlinger::init, the mDisplaySurface in DisplayDevice comes from the same BufferQueue as EGLSurface, look at the code 

    bq = new BufferQueue(new GraphicBufferAlloc());
    fbs = new FramebufferSurface(…, bq);
    hw = new DisplayDevice(…, fbs, bq, …);
    mDisplays.add(…,hw);

So now EGL can swap to the FrameBufferSurface, because the EGLSurface is the provider end of the same BufferQueue which hosts FrameBufferSurface.
The layer represented by the frame buffer target is tracked by DisplayDevice::framebufferTarget, ANativeWindow and ANativeWindowBuffer is the interface used by EGL to access the buffer allocated in SurfaceFlinger. Basically flow is:

swap to DisplayDevice::mSurface, DisplayDevice::mNativeWindow influenced too -> BufferQueue::queueBuffer -> FramebufferSurface::onFrameAvailable -> HWComposer::fbPost -> HWComposer::setFramebufferTarget -> get framebuffer target fram handle parameter -> disp.framebufferTarget.handle = disp.fbTargetHandle;

    Stack trace: FramebufferSurface::onFrameAvailable (24 frames)
    ...
    #02  pc 0002dffa  /system/lib/libgui.so (android::BufferQueue::ProxyConsumerListener::onFrameAvailable()+74)
    #03  pc 00030ca9  /system/lib/libgui.so (android::BufferQueue::queueBuffer(int, android::IGraphicBufferProducer::QueueBufferInput const&, android::IGraphicBufferProducer::QueueBufferOutput*)+2425)
    #04  pc 000481cf  /system/lib/libgui.so (android::Surface::queueBuffer(ANativeWindowBuffer*, int)+559)
    #05  pc 000470c9  /system/lib/libgui.so (android::Surface::hook_queueBuffer(ANativeWindow*, ANativeWindowBuffer*, int)+41)
    ...
    #17  pc 0001e895  /system/lib/libutils.so (android::Looper::pollOnce(int, int*, int*, void**)+53)
    #19  pc 00022a67  /system/lib/libsurfaceflinger.so (android::SurfaceFlinger::run()+39)
    ...
    #22  pc 00000a02  /system/bin/surfaceflinger


To get clear understanding of the process and the role, reference to this picture:

![surfaceflinger as consumer and producer](/images/post/surfaceflinger-as-consumer-and-producer.png)

## Summary 
1. Surface dequeueBuffer eventually works on BuffferQueue in SF, Surface is provider, SurfaceFlingerConsumer is consumer
2. GraphicBuffer in allocated for Surface to draw.
3. Surface queueBuffer eventually works on BufferQueue in SF, Layer::onFrameAvailable called.
4. SurfaceFlingerConsumer::updateTexImage bind EGLImage as texture
5. Layer::onDraw draws to the BufferQueue of FramebufferSurface, here egl is provider, FramebufferSurface is consumer
6. FramebufferSurface::onFrameAvailable -> HWComposer::fbPost -> HWComposer::setFramebufferTarget
7. All layers prepared, sent to HWC

