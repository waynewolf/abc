---
title: surfaceflinger and hwcomposer
date: 2014-07-10 20:26:03
tags:
- android
- surfaceflinger
- hwcomposer
---

## Introduction
In the beginning, people believes that GPU composition is fast and power effiecent compared with CPU composition. Then, people find that in most usage scenarios, GPU can be kept sleep with the introduction of more display planes. This is where HWComposer comes into play. It is linked into SurfaceFlinger as a library, and works as a judge to decide when to offload work from GPU to display hardware.

<!--more-->
## General flow

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

### Recap of the draw-a-frame flow 
* SurfaceFlinger::setUpHWComposer
	* beginFrame
	* hwc prepare
	* prepareFrame
* DisplayDevice::swapBuffers
	* eglSwapBuffer
	* advanceFrame
* DisplayDevice::onSwapBuffers
	* hwc set
	* onFrameComitted

## hwc prepare
One of the major interfaces of hwc, decision is made in this phase if a layer should belended by GPU or display. Generally speaking, a layer that can be blended by display should be marked as HWC_OVERLAY, other layers marked as HWC_FRAMEBUFFER, though other complicated cases exist.

## hwc set
Another major interface of hwc, hwc set callback is used to replace eglSwapBuffers for frames with mixed GLES and HWC layers.
Before hwc set happens, GLES layers already composed to framebuffer target, while not the other layers marked as HWC_OVERLAY. These HWC_OVERLAY layers and framebuffer tareget will be sent to hardware for blend. On my platform, in the parameters of hwc set, the Displays is 3，which includes 0 (primary display)，1(external display - like HDMI)，2(virtual display), mLists contains all the layers of all displays. The layers in a display is called layer
stack.

## Events from HWC to SF
Setup SF callback to HWC in HWComposer constructor

	mHwc->registerProcs(mHwc, &mCBContext->procs);

hotplug:

	HWC -> HWComposer::hook_hotplug -> HWComposer::hotplug -> SurfaceFlinger::onHotplugReceived

hardware vsync:

	HWC -> HWComposer::hook_vsync -> HWComposer::vsync -> SurfaceFlinger::onVSyncReceived

Do not confuse these two callbacks with similar functions in EventThread, like EventThread::onHotplugReceived and EventThread::onVsyncEvent, these functions are used to notify client. 

## How layer maps to plane
The process that software layers map to display plane is driver and framework dependent, and different between implementations. PVR and GEN are lot diffrent. On PVR，DRM pageflip ioctl is not used, dc_plane_ctx structure is allocated instead，specific dc_device_post function is called to parse parameter to kernel. No more detailed explanation here, but for GEN which uses drm and i915 open source implementation, more articles will be written later.


