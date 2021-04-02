# ffmpeg-on-clear-linux
Run [FFmpeg](https://ffmpeg.org/) on [Clear Linux](https://clearlinux.org/) including H.264 and vp9 hardware acceleration in Firefox.

This is an automation How-To repository for building FFmpeg and minimum dependencies. My motivation is wanting hardware acceleration in Firefox. Thank you, @xtknight for the vp9 acceleration bits.

## What's Included
```text
build-all  Top-level script for building dependencies and FFmpeg.
extras     Complementary YouTube player for testing nvdec/nvenc.
localenv   Set your GPU's CUDA compute capability here.
patches    Various patches for vdpau-driver 0.7.4.
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

I'm hoping that the build process succeeds for you as it does for me. However, I may have a bundle installed that's missing in the ```000-install-dependencies``` script. Please reach out if that is the case. The Media SDK is included in the FFmpeg build for Intel hardware, but not yet tested what else is needed on that platform. It may be documenting the value to use for ```LIBVA_DRIVER_NAME```.

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
The following is my Firefox config (running X11). Adjust LIBVA and VDPAU variables accordingly for Intel or AMD GPUs. If running Wayland, replace ```MOZ_DISABLE_WAYLAND``` and ```MOZ_X11_EGL``` with:
```bash
export MOZ_ENABLE_WAYLAND=1
```


```bash
$ cat ~/.config/firefox.conf

export LIBVA_DRIVERS_PATH=/usr/lib64/dri
export LIBVA_DRIVER_NAME=vdpau
export VDPAU_DRIVER=nvidia

export MOZ_DISABLE_WAYLAND=1
export MOZ_X11_EGL=1

export MOZ_DISABLE_RDD_SANDBOX=1
export MOZ_ACCELERATED=1
export MOZ_USE_XINPUT2=1
export MOZ_WEBRENDER=1

export LD_LIBRARY_PATH=/usr/local/lib:/opt/nvidia/lib64
```

## Firefox Settings
Please find the minimum settings applied via ```about:config``` to enable hardware acceleration. The ```media.rdd-ffmpeg.enable``` setting must be enabled for h264ify to work with FFmpeg also supporting vp9. Basically, choose to play videos via the h264ify extension or the vp9 format by disabling h264ify and enjoy beyond 1080P.
```text
gfx.webrender.all                            true
gfx.webrender.enabled                        true
gfx.xrender.enabled                          false

layers.acceleration.force-enabled            false

media.ffmpeg.dmabuf-textures.enabled         true
media.ffmpeg.vaapi.enabled                   true
media.ffvpx.enabled                          false

media.rdd-vpx.enabled                        false
media.rdd-ffvpx.enabled                      false
media.rdd-ffmpeg.enabled                     true
media.av1.enabled                            false
```

To play HDR videos, see ```youtube-play``` found in the extras folder.
