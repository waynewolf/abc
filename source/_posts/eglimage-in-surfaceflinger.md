---
title: eglimage in surfaceflinger
date: 2014-05-017 20:01:55
tags:
- android
- egl
---

In android, the application UI is used as an OpenGL ES texture, and composed by SurfaceFlinger to form final image on display. To understand how the application UI is sent to OpenGL, we have to start from some egl/oes extensions.

## EGL_KHR_image_base & EGL_KHR_image

The spec says:

> This extension defines a new EGL resource type that is suitable for
> sharing 2D arrays of image data between client APIs, the EGLImage.
> Although the intended purpose is sharing 2D image data, the
> underlying interface makes no assumptions about the format or
> purpose of the resource being shared, leaving those decisions to
> the application and associated client APIs.

This extension is used to share buffers between different client APIs. EGLImage is an opaque handle to a shared resource, presumably a 2D array of image data.

EGL allows sharing context from the same client API (like OpenGL ES), however, in some cases, it may be desireable to share state between different client APIs(OpenGL ES <-> OpenVG). EGLImage is introduced to address this kind of complicated use cases.

EGL can create EGL resources that can be shared between contexts of different client APIs from client API resources like texel arrays in OpenGL ES texture objects. The client API resources called EGLImage source. In contrast, the client API can also create resources from EGLImage, and the created resources are called EGLImage target. Refer to spec for more concepts like EGLImage sibling.

New Types added:

	typedef void* EGLImageKHR;

New Procedures and Functions:

	EGLImageKHR eglCreateImageKHR(
					EGLDisplay dpy,
					EGLContext ctx,
					EGLenum target,
					EGLClientBuffer buffer,
					const EGLint *attrib_list)

    EGLBoolean eglDestroyImageKHR(
					EGLDisplay dpy,
					EGLImageKHR image)

Please reference [EGL_KHR_image_base](https://www.khronos.org/registry/egl/extensions/KHR/EGL_KHR_image_base.txt) for more detailed explanation.

## OES_EGL_image

The spec says:

> This extension provides a mechanism for creating texture and renderbuffer objects sharing storage with specified EGLImage objects.
> The companion EGL_KHR_image_base and EGL_KHR_image extensions provide the definition and rationale for EGLImage objects.

This extension allows EGLImage turned into a texture.

New type added:

    typedef void* GLeglImageOES;

New Procedures and Functions added:

    void EGLImageTargetTexture2DOES(enum target, eglImageOES image)
    void EGLImageTargetRenderbufferStorageOES(enum target, eglImageOES image)

glEGLImageTargetTexture2DOES serves the same purpose as the well known glTexImage2D, but supporting EGLImage. 

Please reference [OES_EGL_image](https://www.khronos.org/registry/gles/extensions/OES/OES_EGL_image.txt) for more detailed explanation.

## OES_EGL_image_external

The spec says:

> This extension provides a mechanism for creating EGLImage texture targets
> from EGLImages.  This extension defines a new texture target,
> TEXTURE_EXTERNAL_OES.  This texture target can only be specified using an
> EGLImage.
> 

This extension defines a new texture target TEXTURE_EXTERNAL_OES, GLSL can use new type samplerExternalOES to sample the external texture. This extension depends on EGL_KHR_image_base or EGL_KHR_image.

GLSL sampler type added:

	samplerExternalOES (when #extension GL_OES_EGL_image_external is used)

Despite the new texture target and sampler type, the intension of this extension is not clear until reading through the spec carefully. I guess the main purpose of this extension is allow driver to support color space converter. The spec says:

> Sampling an external texture will return an RGBA vector in the same
> colorspace as the source image.  If the source image is stored in YUV
> (or some other basis) then the YUV values will be transformed to RGB
> values. (But these RGB values will be in the same colorspace as the
> original image.  Colorspace here includes the linear or non-linear
> encoding of the samples. For example, if the original image is in the
> sRGB color space then the RGB value returned by the sampler will also
> be sRGB, and if the original image is stored in ITU-R Rec. 601 YV12
> then the RGB value returned by the sampler will be an RGB value in the
> ITU-R Rec. 601 colorspace.) The parameters of the transformation
> from one basis (e.g.  YUV) to RGB (color conversion matrix, sampling
> offsets, etc) are taken from the EGLImage which is associated with the
> external texture.  The implementation may choose to do this
> transformation when the external texture is sampled, when the external
> texture is bound, or any other time so long as the effect is the same.
> It is undefined whether texture filtering occurs before or after the
> transformation to RGB.

This extension doesn't support mipmap. The spec says:

> Calling GenerateMipmaps with <target> set to TEXTURE_EXTERNAL_OES results in an INVALID_ENUM

In short, 3D driver is required to implement this color space transformation depending on the EGLImage format. Please reference [OES_EGL_image_external](https://www.khronos.org/registry/gles/extensions/OES/OES_EGL_image_external.txt) for more detailed explanation.

## Android Usage

In android, a specific target EGL_NATIVE_BUFFER_ANDROID is used in eglCreateImageKHR, to create EGLImage from ANativeWindowBuffer.

	eglCreateImageKHR(eglDisplay, EGL_NO_CONTEXT, EGL_NATIVE_BUFFER_ANDROID,
		(EGLClientBuffer)&sSrcBuffer,0);
	glGenTextures(1,&texID);
	glBindTexture(GL_TEXTURE_2D,texID);
	glEGLImageTargetTexture2DOES(GL_TEXTURE_2D,eglSrcImage);

If the format of EGLClientBuffer is YUV, the texture target must be GL_TEXTURE_EXTERNAL_OES. This texture target, as explained early, can only be used in glEGLImageTargetTexture2DOES. It's illegal to use this in gl*Tex*Image*() functions.

This is how OpenGL ES supports YUV format texture, the transformation work is offloaded to EGLImage in egl driver.

## Appendix
All kinds of buffers across the graphic stack:

![all-kinds-of-buffers](/images/post/all-kinds-of-buffers.png "all kinds of buffers")

## Reference

1. <https://www.khronos.org/registry/egl/extensions/KHR/EGL_KHR_image_base.txt>
2. <https://www.khronos.org/registry/gles/extensions/OES/OES_EGL_image.txt>
3. <https://www.khronos.org/registry/gles/extensions/OES/OES_EGL_image_external.txt>
