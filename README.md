# ffmpeg-on-clear-linux

Run [FFmpeg](https://ffmpeg.org/) on [Clear Linux](https://clearlinux.org/) including H.264 and vp9 hardware acceleration in Firefox.

This is an automation How-To repository for building FFmpeg and minimum dependencies. My motivation is wanting hardware acceleration in Chromium and Firefox. Thank you, @xtknight for the initial [vp9](https://github.com/xtknight/vdpau-va-driver-vp9) acceleration bits. Thank you, @xuanruiqi for the [vp9-update](https://github.com/xuanruiqi/vdpau-va-driver-vp9) with Arch Linux patches and additional fixes.

## What's Included

```text
build-all  Top-level script for building dependencies and FFmpeg.
extras     Complementary YouTube player for testing nvdec/nvenc.
localenv   Set your GPU's CUDA compute capability here.
scripts    Contains the individual build/install scripts.
```

## Requirements

Although testing was done using a NVIDIA GPU, the Intel Media SDK is included during the build process. For NVIDIA GPUs, this requires the proprietary driver to be installed under ```/opt/nvidia```. Optionally install CUDA for extra hardware acceleration capabilities. See installation guides [NVIDIA Drivers](https://docs.01.org/clearlinux/latest/tutorials/nvidia.html) and [NVIDIA CUDA Toolkit](https://docs.01.org/clearlinux/latest/tutorials/nvidia-cuda.html).

Set your GPU's [compute capability](https://en.wikipedia.org/wiki/CUDA) in ```localenv```. The file resides at the top-level and is ignored by Git. The GeForce GTX 1660 model supports max ```7.5``` compute capability.
```text
cudaarch="compute_75"  # Turing
cudacode="sm_75"
```

Enable ```ForceCompositionPipeline``` for a better desktop experience, especially when moving/resizing a terminal window while playing a video. This can be done at the device level by adding/editing a file ```/etc/X11/xorg.conf.d/nvidia-device.conf```. Replace ```MODEL_STRING``` with your actual GPU model (i.e. GTX 1660). Finally reboot for the change to take effect.
```text
Section "Device"
    Identifier    "Device0"
    Driver        "nvidia"
    VendorName    "NVIDIA Corporation"
    BoardName     "GeForce MODEL_STRING"
    Option        "ForceCompositionPipeline" "True"
EndSection
```

## Building and Installation

The build and installation is a one-step process.
```bash
$ sudo bash build-all
```

Or become root and run each script individually. Various scripts exit silently depending on whether ```/opt/nvidia/bin/nvidia-settings``` or ```/usr/local/cuda/cuda.h``` exists.
```bash
$ sudo root

cd scripts
./000-install-dependencies
./100-build-nv-codec-headers
./110-build-libvdpau
...
```

The ```builddir``` (once created) serves as a cache folder. Remove the correspondent ```*.tar.gz``` file(s) to re-fetch or re-clone from the internet.

I'm hoping that the build process succeeds for you as it does for me. However, I may have a bundle installed that's missing in the ```000-install-dependencies``` script. Please reach out if that's the case. The Media SDK is included in the FFmpeg build for Intel hardware, but not yet tested what else is needed on that platform. It may be documenting the value to use for ```LIBVA_DRIVER_NAME```.

Remember to add ```/usr/local/bin``` to your ```PATH``` environment variable if not already done.

## x264 and x265 Multilibs

Below ```x264``` supports 8-bits and 10-bits output.
```bash
$ x264 --help | grep "Output bit depth"
Output bit depth: 8/10

$ ffmpeg -hide_banner -h encoder=libx264 | grep "Supported pixel formats"
Supported pixel formats: yuv420p yuvj420p yuv422p yuvj422p yuv444p yuvj444p
nv12 nv16 nv21 yuv420p10le yuv422p10le yuv444p10le nv20le gray gray10le
```

Likewise x265 supports 8-bits, 10-bits and 12-bits output.
```bash
$ x265 --help | grep "Output bit depth"
-D/--output-depth 8|10|12    Output bit depth... Default 8

$ ffmpeg -hide_banner -h encoder=libx265 | grep "Supported pixel formats"
Supported pixel formats: yuv420p yuvj420p yuv422p yuvj422p yuv444p yuvj444p
gbrp yuv420p10le yuv422p10le yuv444p10le gbrp10le yuv420p12le yuv422p12le
yuv444p12le gbrp12le gray gray10le gray12le
```

## Firefox Config

The following is my Firefox config (running XOrg). Adjust LIBVA and VDPAU variables accordingly. For ```vdpauinfo``` to work on Intel graphics, install the ```va_gl``` [driver](https://github.com/i-rinat/libvdpau-va-gl) driver. Change to ```LIBVA_DRIVER_NAME=i965``` and ```VDPAU_DRIVER=va_gl```. If that is not working, try ```LIBVA_DRIVER_NAME=iHD``` or comment out the first 3 export lines, exporting only ```LD_LIBRARY_PATH```.

```bash
$ cat ~/.config/firefox.conf

export LIBVA_DRIVERS_PATH=/usr/lib64/dri
export LIBVA_DRIVER_NAME=vdpau
export VDPAU_DRIVER=nvidia
export LD_LIBRARY_PATH=/opt/nvidia/lib64:/usr/local/lib

if [ $XDG_SESSION_TYPE == wayland ]
then
    export MOZ_ENABLE_WAYLAND=1
elif [ $XDG_SESSION_TYPE == x11 ]
then
    export MOZ_DISABLE_WAYLAND=1
    export MOZ_X11_EGL=1
fi

export MOZ_ACCELERATED=1
export MOZ_DISABLE_RDD_SANDBOX=1
export MOZ_USE_XINPUT2=1
export MOZ_WEBRENDER=1
```

## Firefox Settings

Please find the minimum settings applied via ```about:config``` to enable hardware acceleration. The ```media.rdd-ffmpeg.enable``` setting must be enabled for h264ify to work with FFmpeg also supporting vp9. Basically, choose to play videos via the h264ify extension or the vp9 format by disabling h264ify and enjoy beyond 1080P playback.
```text
gfx.canvas.azure.accelerated                   true
gfx.webrender.all                              true
gfx.webrender.enabled                          true

Enable software render if you prefer to render on the CPU instead of GPU.
This is helpful if you want more GPU availability for playing videos.
gfx.webrender.software                         true

Do not add xrender if missing or set to false or click on the trash icon.
This is a legacy setting that shouldn't be used as it disables WebRender.
gfx.xrender.enabled                            false

Ensure false so to be on a supported code path for using WebRender.
layers.acceleration.force-enabled              false

media.ffmpeg.dmabuf-textures.enabled           true
media.ffmpeg.vaapi-drm-display.enabled         true
media.ffmpeg.vaapi.enabled                     true
media.ffvpx.enabled                            false

media.rdd-ffmpeg.enabled                       true
media.rdd-ffvpx.enabled                        false
media.rdd-process.enabled                      false
media.rdd-vpx.enabled                          false
media.av1.enabled                              false

Enable FFMPEG VA-API decoding support for WebRTC on Linux.
media.navigator.mediadatadecoder_vpx_enabled   true
```

## Run Chromium

This repository [chromium-latest-linux](https://github.com/scheib/chromium-latest-linux) for launching Chromium works great including ```vp9``` video playback.

**Edit run.sh script**

Insert lines exporting ```LIBVA_DRIVERS_PATH```, ```LIBVA_DRIVER_NAME```, ```VDPAU_DRIVER```, and ```LD_LIBRARY_PATH```. For Intel graphics, change to ```LIBVA_DRIVER_NAME=i965``` and ```VDPAU_DRIVER=va_gl```. If that is not working, as noted above try ```LIBVA_DRIVER_NAME=iHD``` or comment out the first 3 export lines, exporting only ```LD_LIBRARY_PATH```.

```bash
#! /bin/bash

export LIBVA_DRIVERS_PATH=/usr/lib64/dri
export LIBVA_DRIVER_NAME=vdpau
export VDPAU_DRIVER=nvidia
export LD_LIBRARY_PATH=/opt/nvidia/lib64:/usr/local/lib

# To rid of the Google API keys are missing notification.
export GOOGLE_API_KEY=no
export GOOGLE_DEFAULT_CLIENT_ID=no
export GOOGLE_DEFAULT_CLIENT_SECRET=no

BASEDIR=$(dirname $0)

$BASEDIR/latest/chrome --user-data-dir="$BASEDIR/user-data-dir" $* &> /dev/null &
```

**First time**

On first launch, go into ```Settings -> Appearance -> Customize fonts``` and change the fonts to your liking. On Clear Linux, Standard font ```Noto Sans```, Serif font ```Noto Serif```, and Sans-serif font ```Noto Sans``` look great.

```bash
$ ./update-and-run.sh
```

**Subsequently**

```bash
$ ./run.sh
```

## HDR Videos

To play HDR videos, see ```youtube-play``` found in the extras folder.

## See Also

[Hardware video acceleration](https://wiki.archlinux.org/title/Hardware_video_acceleration) at Arch Linux.

