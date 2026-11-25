# v4l2-request libVA Backend

> **Fork note:** this fork carries the changes needed to get a working
> setup on the **Banana Pi M1 (Allwinner A20)** with a **7.x kernel**.
> The problems found, their fixes and the tested configuration are in the
> [Allwinner A20 (Banana Pi M1) Addendum](#allwinner-a20-banana-pi-m1-addendum)
> at the end of this file.

## About

This libVA backend is designed to work with the Linux Video4Linux2
Request API that is used by a number of video codecs drivers,
including the Video Engine found in most Allwinner SoCs.

## Status

The v4l2-request libVA backend currently supports the following formats:
* MPEG2 (Simple and Main profiles)
* H264 (Baseline, Main and High profiles)
* H265 (Main profile)

## Instructions

In order to use this libVA backend, the `v4l2_request` driver has to
be specified through the `LIBVA_DRIVER_NAME` environment variable, as
such:

	export LIBVA_DRIVER_NAME=v4l2_request

A media player that supports VAAPI (such as VLC) can then be used to decode a
video in a supported format:

	vlc path/to/video.mpg

Sample media files can be obtained from:

	http://samplemedia.linaro.org/MPEG2/
	http://samplemedia.linaro.org/MPEG4/SVT/

## Technical Notes

### Surface

A Surface is an internal data structure never handled by the VA's user
containing the output of a rendering. Usualy, a bunch of surfaces are created
at the begining of decoding and they are then used alternatively. When
created, a surface is assigned a corresponding v4l capture buffer and it is
kept until the end of decoding. Syncing a surface waits for the v4l buffer to
be available and then dequeue it.

Note: since a Surface is kept private from the VA's user, it can ask to
directly render a Surface on screen in an X Drawable. Some kind of
implementation is available in PutSurface but this is only for development
purpose.

### Context

A Context is a global data structure used for rendering a video of a certain
format. When a context is created, input buffers are created and v4l's output
(which is the compressed data input queue, since capture is the real output)
format is set.

### Picture

A Picture is an encoded input frame made of several buffers. A single input
can contain slice data, headers and IQ matrix. Each Picture is assigned a
request ID when created and each corresponding buffer might be turned into a
v4l buffers or extended control when rendered. Finally they are submitted to
kernel space when reaching EndPicture.

The real rendering is done in EndPicture instead of RenderPicture
because the v4l2 driver expects to have the full corresponding
extended control when a buffer is queued and we don't know in which
order the different RenderPicture will be called.

### Image

An Image is a standard data structure containing rendered frames in a usable
pixel format. Here we only use NV12 buffers which are converted from sunxi's
proprietary tiled pixel format with tiled_yuv when deriving an Image from a
Surface.

---

# Kernel 6.x Port Addendum

The sections below document the state of this tree after the port to
the stable stateless codec API of kernel 6.x (tested against
linux-headers 6.18). The original notes above are kept unchanged.
See DEVLOG-kernel-6.18.md for the full story of the port.

## What the program does

The backend translates VA-API into the V4L2 Request API. The
application talks regular VA-API to libva, and this driver turns each
decode job into V4L2 stateless decoding requests: it fills the codec
control structures (SPS, PPS, slice and decode parameters), copies the
bitstream into V4L2 output buffers, ties everything together with a
media request and queues it to the kernel driver. The hardware decoder
writes the decoded frame into the capture buffer that backs the VA
surface.

Any V4L2 stateless decoder can be driven this way. Tested kernel
drivers are cedrus (Allwinner) and rpivid (Raspberry Pi 4/400 HEVC).
On the Pi, HEVC decoding has been verified bit-exact against software
decoding at 720p and 1080p. H264 and MPEG2 still compile and probe,
but the Pi has no stateless hardware for them.

Note for Raspberry Pi 4/400 users: only HEVC gets hardware decoding
through this backend on a Pi. The format list above describes what
the backend can translate, but it only works with stateless decoders
(the V4L2 Request API), because that is the model VA-API maps onto:
the application parses the bitstream and hands pre-parsed parameters
to the hardware for every frame. The only stateless decoder on the Pi
is rpivid (HEVC). The Pi's H264 decoder (bcm2835-codec) is stateful,
meaning the hardware parses the bitstream itself, which is a
different V4L2 interface this backend cannot drive. For H264 on the
Pi use the stateful decoder directly instead, for example FFmpeg's
h264_v4l2m2m decoder or mpv with --hwdec=v4l2m2m-copy. The full
MPEG2, H264 and H265 list applies to boards whose kernel driver
exposes all three as stateless decoders, such as Allwinner (cedrus).

Known limitations:
* Frames split into multiple slices only submit the parameters of the
  last slice, so multi-slice streams may show artifacts.
* HEVC scaling lists (IQ matrices) are not passed to the kernel yet.
  VA-API provides them in diagonal scan order while V4L2 expects
  raster order.
* Zero-copy export of SAND128 (Raspberry Pi) surfaces through DMA-BUF
  is untested. Use the copy path (vaGetImage / vaDeriveImage), which
  untiles in software and is known good.

## Building and installing with meson

The project needs libva and libdrm development files plus the kernel
uapi headers from a 6.x kernel:

	meson setup build
	ninja -C build
	sudo ninja -C build install

This installs `v4l2_request_drv_video.so` into the libva driver
directory of the chosen prefix, for example `/usr/local/lib/dri`.

## How to use

The driver is selected through environment variables. The first one
names the driver, the second one is only needed when the install
prefix is not on the default libva search path:

	export LIBVA_DRIVER_NAME=v4l2_request
	export LIBVA_DRIVERS_PATH=/usr/local/lib/dri

By default the backend opens `/dev/video0` and `/dev/media0`. Most
boards register the decoder elsewhere, so point it at the right nodes.
On a Raspberry Pi 4/400 the rpivid HEVC decoder is usually:

	export LIBVA_V4L2_REQUEST_VIDEO_PATH=/dev/video19
	export LIBVA_V4L2_REQUEST_MEDIA_PATH=/dev/media0

To find the right nodes on your board, check `v4l2-ctl --list-devices`
(from v4l-utils) or `dmesg | grep -i "registered as"`. The user also
needs permission to open the video and media nodes, which usually
means being in the `video` group:

	sudo usermod -aG video $USER

Then any VA-API capable player works:

	mpv --hwdec=vaapi-copy path/to/video.mkv
	ffplay -hwaccel vaapi path/to/video.mkv

## How to verify

Three levels of checking, from quick to thorough:

1. Driver loads and reports profiles:

		vainfo

	Success looks like `Driver version: v4l2-request` followed by the
	profile list (for example `VAProfileHEVCMain : VAEntrypointVLD`).
	A `vaInitialize failed` error almost always means the environment
	variables above are not set in the current shell.

2. Hardware actually decodes. Decode a file through the driver and
   watch for errors:

		ffmpeg -hwaccel vaapi -hwaccel_device /dev/dri/card0 \
			-i test.mkv -f null -

	The backend logs its V4L2 calls to stderr (s_fmt, create_bufs,
	streamon, dqbuf). If a frame fails in the kernel driver, the
	dequeue reports an error flag and the kernel log explains why,
	so also check `sudo dmesg`.

3. Output is correct. Compare hardware and software decodes frame by
   frame, the hashes must be identical:

		ffmpeg -hwaccel vaapi -i test.mkv -vf format=nv12 -f framemd5 hw.md5
		ffmpeg -i test.mkv -vf format=nv12 -f framemd5 sw.md5
		diff hw.md5 sw.md5 && echo "bit-exact"

---

# Allwinner A20 (Banana Pi M1) Addendum

## Acknowledgements and tested kernels

Credit goes to cn8001 for the port of this backend to the stable
stateless codec API. The original code was written in 2019 against the
staging H.264/HEVC controls. It now builds cleanly against 6.x kernel
headers, and the H.264 and H.265 paths were rewritten for the stable
control structures (`V4L2_CID_STATELESS_*`, the separate
`PRED_WEIGHTS` control, reference pictures identified by timestamp).
That is the hard part, and it is what makes this tree usable on a
modern kernel at all.

The result is not limited to 6.x: the same tree has also been built,
loaded and used to decode H.264 and MPEG2 on a machine running a
**7.3 kernel**.

Running it on real hardware is a different matter, though. On the
Allwinner A20 (Banana Pi M1) several problems showed up, from crashes
at start-up to wrong pictures and stuttering playback. They are
documented below, each with its cause and fix.

The port above builds cleanly and also works with a 7.x kernel, but
running it on an Allwinner A20 (Banana Pi M1, Cedrus video engine)
exposed several problems. This section records each one with its
symptom, its cause and the fix. Problems 1 and 2 stop the decoder at
start-up, 3 and 4 are decoding bugs, and 5 is a performance problem.

## How this work was done

The investigation, the fixes and the documentation in this addendum were
carried out with the assistance of an AI assistant, **Claude** (by
Anthropic). Claude helped analyse the logs and measurements, proposed
the changes and drafted the text. The changes were applied, built and
tested on the real hardware by the maintainer of this fork, and the
results reported here are the ones observed on that board.

## Tested configuration

* Board: Banana Pi M1 (Allwinner A20, dual Cortex-A7, 1 GB RAM), 32-bit
  ARM userspace (Debian armhf)
* Kernel: 7.3, with the Cedrus change described in problem 5
* libva 2.22, display through labwc (Wayland)
* Players: ffmpeg with VA-API, and `mpv --hwdec=vaapi-copy`
* Codecs tested: **H.264** and **MPEG2**. `vainfo` lists the MPEG2 and
  H264 profiles on this board.

Cedrus writes frames in a tiled NV12 format (`ST12`, 32x32 tiles).
This backend converts it to linear NV12 in `tiled_to_planar()`. On
32-bit ARM that function is the NEON assembly in `tiled_yuv.S`. The C
version in `tiled_yuv.c` is built only for aarch64.

## Problems and fixes

### 1. SIGBUS on the first frame (bad CAPTURE size)

Symptom: the application dies with SIGBUS in `tiled_to_planar()` on
the first frame. The debug log shows the CAPTURE format being clamped:

	s_fmt type=1 fourcc=ST12 640x480 -> 16x32

Cause: `RequestCreateSurfaces2()` negotiates the CAPTURE format before
the OUTPUT queue is configured, which only happens later in
`RequestCreateContext()`. Cedrus then clamps CAPTURE to its minimum
size, the surfaces are created with that clamped format, and
`tiled_to_planar()` crashes on the first frame.

Fix: save the profile in `RequestCreateConfig()`, and at the start of
`RequestCreateSurfaces2()` do `S_FMT` on the OUTPUT queue (bitstream
pixel format for that profile, real frame size) before touching
CAPTURE. After the fix the log shows:

	s_fmt type=2 fourcc=S264 640x368 -> 640x368
	s_fmt type=1 fourcc=ST12 640x368 -> 640x384

### 2. EBUSY when creating the context

Symptom, after fix 1:

	Unable to set format for type 2: Device or resource busy

Cause: `RequestCreateContext()` sets the OUTPUT format a second time.
By then the format has already been set in `RequestCreateSurfaces2()`
and the CAPTURE surfaces exist, and V4L2 drivers typically refuse to
change a format once buffers have been created.

Fix: in `RequestCreateContext()`, skip the OUTPUT `S_FMT` if the OUTPUT
queue was already configured.

### 3. Wrong luma in P and B frames (8x8 scaling lists)

Symptom: I frames decode correctly, but from the first P frame the
picture is wrong. Compared with software decoding, the chroma plane is
bit-exact while the luma plane differs on most pixels (about 27% of the
bytes equal on the frame tested), with streaks and blocks in moving
areas. In mpv the picture was also black and white and full of
artifacts, and both problems went away with this fix.

Cause: `h264_va_matrix_to_v4l2()` copied the inter-Y 8x8 scaling list
into `scaling_list_8x8[3]`. That index belongs to the old staging
API. In the stable stateless API the 8x8 lists are ordered intra Y,
inter Y, then the Cb/Cr lists, so the kernel read an inter-Y list that
was all zeros. A zero matrix removes the residual of 8x8 inter blocks,
which only affects luma in P and B frames.

The backend also set `V4L2_H264_PPS_FLAG_SCALING_MATRIX_PRESENT` on
every frame. Cedrus reads the scaling lists (`scaling_list_8x8[0]` and
`[1]`) only when that flag is set, and otherwise selects the hardware's
default matrix. The forced flag therefore made it use the broken matrix
even on streams that never sent one.

Fix: copy `ScalingList8x8[0]` to `scaling_list_8x8[0]` and
`ScalingList8x8[1]` to `scaling_list_8x8[1]`. The remaining slots
only concern chroma in 4:4:4 and stay zero. Filling all six slots in a
loop, alternating the two luma lists, works equally well.

Alternative fix: stop forcing `V4L2_H264_PPS_FLAG_SCALING_MATRIX_PRESENT`
and skip the call to `h264_va_matrix_to_v4l2()` altogether. Without the
flag Cedrus ignores the scaling-matrix control and uses its default
matrix, and the backend saves the work of building the matrix on every
frame. The limit is that matrices carried by the stream are never
passed on, so streams with their own scaling lists will not decode
correctly with this variant. Use the fix above for them.

All three variants (copying to `[0]` and `[1]`, the six-slot loop, and
skipping the matrix call) were tested on the board and each one removed
the artifacts on the streams tested.

After the first fix, the 640x368 stream was compared frame by frame
with software decoding and matched. On the 1920x818 stream the picture and
colours were only checked by eye.

### 4. Segmentation fault when the DPB is full

Symptom: crash in the decoding thread, with this backtrace:

	memset()
	dpb_clear_entry (entry=0x0, reserved=true)   h264.c
	h264_set_controls ()                          h264.c

Cause: `dpb_find_entry()` returns NULL when all 16 DPB slots are valid
and in use. `h264_set_controls()` passed that NULL to
`dpb_clear_entry()` and later to `dpb_insert()`, and both dereferenced
it. The `used` flags are only cleared in `dpb_update()`, which runs
after that search. A 640x368 stream hit this: the DPB was reported full
73 times in a 299-frame file.

Fix: `dpb_clear_entry()` returns immediately for a NULL entry, and
`dpb_insert()` returns without writing when no slot is free. A picture
that was not inserted is added again by `dpb_update()` on the next
frame if it is still referenced. A warning is printed when it happens.

After the fix, the stream decodes to the end and all 299 frames matched
software decoding. Only this one stream has been tested, so other
streams may expose more DPB corner cases.

### 5. Stuttering playback: reading CAPTURE buffers is very slow

Symptom: playback stutters while the CPU is not saturated (about 110%
in `top`, with two cores). `perf` shows about 79% of the time in
`tiled_to_planar()`, and almost all of it on the instruction right
after the first `vld1` load, which means the core is waiting for
memory. Measured on a 1920x818 stream:

* the same routine on normal memory took about 4.7 ms per luma plane
* inside the driver it took about 37.7 ms per call
* a plain `memcpy()` of one decoded plane took about 33 ms on average

Cause: V4L2 buffers are allocated as coherent memory by default. On
this platform they appear to be mapped uncached in userspace, which is
consistent with the timings above, so every CPU read goes to DRAM and
the CPU reads the whole frame to convert it. CPU usage in `top` is a
poor guide here: a core stalled on loads still counts as busy, so the
load looked the same with and without the fix below.

Fix, in two parts that must be used together to get the speedup.

Userspace (this repository): in `v4l2_create_buffers()`, ask for
cacheable buffers on the CAPTURE queue:

	if (buffers.format.type == V4L2_BUF_TYPE_VIDEO_CAPTURE ||
	    buffers.format.type == V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE)
		buffers.flags = V4L2_MEMORY_FLAG_NON_COHERENT;

The buffers that are actually used come from `VIDIOC_CREATE_BUFS`.
Setting the flag only on a `VIDIOC_REQBUFS` call made no difference.

Kernel (Cedrus): the CAPTURE queue must allow cache hints, otherwise
the kernel silently clears the flag (`VIDIOC_REQBUFS` reports no cache
hints). In `cedrus_queue_init()` in
`drivers/staging/media/sunxi/cedrus/cedrus_video.c`:

```diff
 	dst_vq->dev = ctx->dev->dev;
+	dst_vq->allow_cache_hints = 1;
```

Without the kernel change the backend still works, only as slowly as
before. Cedrus does not set this field in the mainline kernel at the
time of writing.

Result on the same stream, average per call:

* `memcpy()` of a decoded plane: about 33 ms down to about 4 ms
* `tiled_to_planar()` reading directly from the buffer: about 37.7 ms
  down to about 9 ms
* playback with `mpv --hwdec=vaapi-copy` under Wayland became
  reasonably smooth. mpv still counted some dropped frames in one run,
  while the debug output was enabled.

After the change, the 640x368 stream was compared frame by frame with
software decoding and matched. The 1920x818 stream, on which the timings
above were taken, was only checked by eye. The 9 ms is still higher than
the 4.7 ms measured on normal memory, and the reason has not been
established.

## Other notes for the A20

* The backend prints `DEBUG` lines on stderr for every `s_fmt`,
  `create_bufs`, `dqbuf` and stream on/off call, and there is no switch
  to turn them off. Writing them to a terminal costs frames during
  playback. The ones called repeatedly during playback, `create_bufs`
  and `dqbuf` in `src/v4l2.c`, can be commented out to save execution
  time. Otherwise redirect stderr (`2>/dev/null`) or use
  `mpv --no-terminal` when judging smoothness.
* Use `vaapi-copy`, which copies the frame to normal memory. Without
  the copy, `mpv --hwdec=v4l2request --vo=gpu` showed only a blue
  window here, so the tiled frames do not seem usable by the GPU path.
  Playback straight from the decoder works outside a compositor, with
  `mpv --hwdec=v4l2request --vo=drm` and the display overlay, but that
  uses FFmpeg's own `v4l2request` support and not this backend.
* ffmpeg must be built with `--enable-vaapi` to use this backend. The
  error `Device creation failed: -12` was caused by a build without it.
* For `vainfo` and `ffmpeg` to find the driver in a console or SSH
  session, `LIBVA_DRIVER_NAME` (and `LIBVA_DRIVERS_PATH` if needed) must
  be set in that shell.

## Known limitations on the A20

* H.264 has been tested on two streams (640x368 and 1920x818), and
  MPEG2 on a few files. `vainfo` lists no HEVC profile on this board.
* Streams with several slices per frame are affected by the existing
  limitation listed in the 6.x addendum.
* The fixes for problems 1 and 2 changed the order in which the formats
  are negotiated. Other Cedrus-based SoCs have not been retested.
