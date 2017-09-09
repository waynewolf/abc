---
title: libdrm samples
date: 2012-09-05 20:13:34
tags:
- drm
---

## Background 
DRM(Direct Rendering Manager) is the kernel part of DRI(Direct Rendering Infrastructure) originally designed for Linux X display server. In this artical, we focus on how to use libdrm.

DRM mainly has two parts:

* KMS: kernel mode setting
* GEM: Graphics Execution Manager, buffer management

DRM major concepts:

* Framebuffer: Memory infomation such as width, height, depth, bpp, pixel format, etc.
* CRTC: Mode information, resolution, depth, polarity, porch, refresh rate, etc
* Encoder: convert digital signal from CRTC to appropriate analog level, eDP, MIPI, ...
* Connector: Physical connector like HDMI，DVI-D，VGA，S-Video

## librdrm

DRM exports API through ioctl, libdrm is a user mode library to wrap these ioctls, please reference xf86drm.h in libdrm for detailed explanation. The general steps to use the libdrm are:

* open/drmOpen /dev/dri/cardN device node
* call drmModeGetResources, get all the drmModeRes resources. Resources include all the fb, crtc, encoder, connector, etc.
* loop through drmModeRes structure, call drmModeGetConnector, get the first connected connector(DRM_MODE_CONNECTED)
* drmModeConnector stores all the supporting mode, choose one from them.
* loop through drmModeRes，call drmModeGetEncoder. If the encoder matches with the selected mode, save the drmModeModeInfo for later use.
* create kms_driver, create buffer object，get the pitch of the BO，and map the BO to user space.
* draw on the BO using cairo or whatever graphic toolbox you like.
* get original display mode by calling drmModeGetCrtc, this will be used after program exit to restore the original mode.
* get frame buffer ID by calling drmModeAddFB, whose argument is BO handle.
* call drmModeSetCrtc with frame buffer ID, the BO attached with the FB is outputed to display.

## a basic sample
The code snippet that follows these steps:

	fd = open("/dev/dri/card0", O_RDWR);
	resources = drmModeGetResources(fd);

Open drm device, get drmModeRes based on fd. drmModeRes is the central point for following operations, incuding searching connectors and encoders. This picture shows some of the structures in libdrm. The arrows are data flow.

![drm-kms-core-data-structures](/assets/drm-kms-core-data-structures.png "drm kms core data structures")

	for(i=0; i < resources->count_connectors; ++i){ 
		connector = drmModeGetConnector(fd, resources->connectors[i]);
		if(connector != NULL){
			fprintf(stderr, "connector %d found\n", connector->connector_id);
			if(connector->connection == DRM_MODE_CONNECTED
				&& connector->count_modes > 0)
				break;
			drmModeFreeConnector(connector);
		}
		else
			fprintf(stderr, "get a null connector pointer\n");
	}
	if(i == resources->count_connectors){
		fprintf(stderr, "No active connector found.\n");
		goto free_drm_res;
	}
	mode = connector->modes[0];
	fprintf(stderr, "(%dx%d)\n", mode.hdisplay, mode.vdisplay);

Search resource->connector array，get the first connected( DRM_MODE_CONNECTED ) connector. Get drmModeInfo by connector->modes[0].

	for(i=0; i < resources->count_encoders; ++i){ 
		encoder = drmModeGetEncoder(fd, resources->encoders[i]);
		if(encoder != NULL){
			fprintf(stderr, "encoder %d found\n", encoder->encoder_id);
			if(encoder->encoder_id == connector->encoder_id);
				break;
			drmModeFreeEncoder(encoder);
		} else
			fprintf(stderr, "get a null encoder pointer\n");
	}
	if(i == resources->count_encoders){
		fprintf(stderr, "No matching encoder with connector, shouldn't happen\n");
		goto free_drm_res;
	}

In all encoders (resource->encoders array), search the matched encoder with connected connector.

	kms_create(fd, &kms_driver);
	unsigned bo_attribs[] = { 
		KMS_WIDTH,   mode.hdisplay,
		KMS_HEIGHT,  mode.vdisplay,
		KMS_BO_TYPE, KMS_BO_TYPE_SCANOUT_X8R8G8B8,
		KMS_TERMINATE_PROP_LIST
	};
	kms_bo_create(kms_driver, bo_attribs, &kms_bo);
	kms_bo_get_prop(kms_bo, KMS_PITCH, &pitch);

create kms_driver, create BO(buffer object), get the pitch of the BO.

	kms_bo_map(kms_bo, &map_buf);
	draw_buffer(map_buf, mode.hdisplay, mode.vdisplay, pitch); 

map BO to user space, map_buf stores the memory address.

	orig_crtc = drmModeGetCrtc(fd, encoder->crtc_id); 

get original display mode by calling drmModeGetCrtc, this is used to restore display mode after program exits.

	kms_bo_get_prop(kms_bo, KMS_HANDLE, &bo_handle); 
	drmModeAddFB(fd, mode.hdisplay, mode.vdisplay, 24, 32, pitch, bo_handle, &fb_id);
	drmModeSetCrtc(  
			fd, encoder->crtc_id, fb_id, 
			0, 0, 	/* x, y */ 
			&connector->connector_id, 
			1, 		/* element count of the connectors array above*/
			&mode);

get BO handle, call drmModeAddFB, get Frame Buffer id (fb_id). set display mode by calling drmModeSetCrtc，output the FB content(map_buf) associated with the BO.

	getchar()
	ret = drmModeSetCrtc(fd, orig_crtc->crtc_id, orig_crtc->buffer_id,
					orig_crtc->x, orig_crtc->y,
					&connector->connector_id, 1, &orig_crtc->mode);

wait for user input, restore the original display mode and quit the program.

## page flip
Above sample draws on the same BO when it is displayed, thus it will probably make display flicker. Page flip use double buffer mechanisam, can be used to avoid flicker. Two frame buffers are created, each associated a BO. The BO being drawn is not the same BO being displayed. The application maintains two frame buffers. Picture is drawn on current frame buffer, and drmModePageFlip is called to send another frame buffer to crtc when vblank comes. Two frame buffers are switched in this way.

In this example, select is used to monitor multiple fds.

	struct termios old_tio, new_tio; 
	tcgetattr(STDIN_FILENO,&old_tio);
	new_tio = old_tio;
	new_tio.c_lflag &= (~ICANON & ~ECHO);
	tcsetattr(STDIN_FILENO, TCSANOW, &new_tio);

close line buffer of stdin in order for the program to receive any key without input enter.

	while(1){
		struct timeval timeout = { 
			.tv_sec = 3, 
			.tv_usec = 0 
		};
		fd_set fds;

		FD_ZERO(&fds);
		FD_SET(STDIN_FILENO, &fds);
		FD_SET(fd, &fds);
		ret = select(max(STDIN_FILENO, fd) + 1, &fds, NULL, NULL, &timeout);

		if (ret <= 0) {
			continue;
		} else if (FD_ISSET(STDIN_FILENO, &fds)) {
			char c = getchar();
			if(c == 'q' || c == 27)
				break;
		} else {
			/* drm device fd data ready */
			ret = drmHandleEvent(fd, &evctx);
			if (ret != 0) {
				fprintf(stderr, "drmHandleEvent failed: %s\n", strerror(errno));
				break;
			}
		}
	}

The main loop of the program. stdin is put into monitor list of select. Whenever terminal has input, select return ready. ESC or 'q' is used to quit the program. We use drmHandleEvent to handle event on the drm device. evctx is an instance of drmEventContext structure, evctx.page_flip_handler points to the page flip handler.

	void page_flip_handler(int fd, unsigned int frame,
		  unsigned int sec, unsigned int usec, void *data)
	{
		struct flip_context *context;
		unsigned int new_fb_id;
		struct timeval end;
		double t;

		context = data;
		if (context->current_fb_id == context->fb_id[0])
			new_fb_id = context->fb_id[1];
		else
			new_fb_id = context->fb_id[0];
			
		drmModePageFlip(fd, context->crtc_id, new_fb_id,
				DRM_MODE_PAGE_FLIP_EVENT, context);
		context->current_fb_id = new_fb_id;
		context->swap_count++;
		if (context->swap_count == 60) {
			gettimeofday(&end, NULL);
			t = end.tv_sec + end.tv_usec * 1e-6 -
				(context->start.tv_sec + context->start.tv_usec * 1e-6);
			fprintf(stderr, "freq: %.02fHz\n", context->swap_count / t);
			context->swap_count = 0;
			context->start = end;
		}
	} 

page_flip_handler get current fb id, get next fb id, call drmModePageFlip. The next fb is set to crtc, frame buffer is swtiched.

	 ret = drmModeSetCrtc(fd, orig_crtc->crtc_id, orig_crtc->buffer_id,
					orig_crtc->x, orig_crtc->y,
					&connector->connector_id, 1, &orig_crtc->mode); 
	tcsetattr(STDIN_FILENO,TCSANOW,&old_tio); 

cleanup, restore work after program exists. tcsetattr is used to restore original terminal setting. 

## Appendix

[kms-pageflip.c](/assets/kms-pageflip.c)
