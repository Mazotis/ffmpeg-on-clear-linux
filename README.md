# ffmpeg-on-clear-linux

Run [FFmpeg](https://ffmpeg.org/) on [Clear Linux](https://clearlinux.org/) including H.264 and VP9 hardware acceleration in Firefox, Google Chrome, and Vivaldi. Chromium can decode VP9, but not yet the media H.264-ACC format.

This is an automation **how-to** for building FFmpeg and minimum dependencies. My motivation is nothing more than wanting hardware acceleration during video playback. Who doesn't want that? Thank you, @xtknight for the initial [VP9](https://github.com/xtknight/vdpau-va-driver-vp9) acceleration bits. Thank you also, @xuanruiqi for the [VP9-update](https://github.com/xuanruiqi/vdpau-va-driver-vp9) to include additional fixes.

* [What's included](#whats-included)
* [Requirements](#requirements)
* [Building and installation](#building-and-installation)
* [x264 and x265 multilibs](#x264-and-x265-multilibs)
* [Determine the VAAPI driver to use](#determine-the-vaapi-driver-to-use)
* [Firefox config](#firefox-config)
* [Firefox settings](#firefox-settings)
* [Chromium installation and configuration](#chromium-installation-and-configuration)
* [Google Chrome installation and run script](#google-chrome-installation-and-run-script)
* [Vivaldi installation and run script](#vivaldi-installation-and-run-script)
* [How can I make sure hardware acceleration is working?](#how-can-i-make-sure-hardware-acceleration-is-working)
* [HDR videos](#hdr-videos)
* [See also](#see-also-annoucement-at-clear-linux-plus-wikis-at-arch-linux)

------

## What's included

```text
build-all  Top-level script for building dependencies and FFmpeg.
extras     Complementary YouTube player for testing nvdec/nvenc.
localenv   Set your NVIDIA GPU's CUDA compute capability here.
scripts    Contains individual build-install scripts.
```

## Requirements

Although testing was done using a NVIDIA GPU, the Intel(R) Media SDK is included during the build process. For NVIDIA GPUs, this requires the proprietary driver to be installed under ```/opt/nvidia```. Well, that's the recommended way in Clear Linux. Optionally install CUDA for extra hardware acceleration capabilities. See installation guides [NVIDIA Drivers](https://docs.01.org/clearlinux/latest/tutorials/nvidia.html) and [NVIDIA CUDA Toolkit](https://docs.01.org/clearlinux/latest/tutorials/nvidia-cuda.html).

Set your GPU's [compute capability](https://en.wikipedia.org/wiki/CUDA) in ```localenv```. The file resides at the top-level and is ignored by Git. For example, the GeForce GTX 1660 model supports max ```7.5``` compute capability. Omit this step is using a non-NVIDIA GPU.

```text
cudaarch="compute_75"  # Turing
cudacode="sm_75"
```

Optionally, enable ```ForceCompositionPipeline``` for a better desktop experience, especially when moving-resizing a terminal window while playing a video. This can be done at the device level by adding-or-editing a file ```/etc/X11/xorg.conf.d/nvidia-device.conf```. Replace ```MODEL_STRING``` with your actual GPU model (i.e. GTX 1660). Finally reboot for the change to take effect.

```text
Section "Device"
    Identifier    "Device0"
    Driver        "nvidia"
    VendorName    "NVIDIA Corporation"
    BoardName     "GeForce MODEL_STRING"
    Option        "ForceCompositionPipeline" "True"
EndSection
```

## Building and installation

The build and installation is a one-step process.

```bash
$ sudo bash build-all
```

Or become root and run each script individually. Be sure to run ```000-install-dependencies``` first if choosing this path. Various scripts exit silently depending on whether ```/opt/nvidia/bin/nvidia-settings``` or ```/usr/local/cuda/cuda.h``` exists.

```bash
$ sudo root

cd scripts
./000-install-dependencies
./100-build-nv-codec-headers
./110-build-libvdpau
...
```

The ```builddir``` (once created) serves as a cache folder. Remove the correspondent ```*.tar.gz``` file(s) to re-fetch or re-clone from the internet.

I'm hoping that the build process succeeds for you as it does for me. However, I may have a bundle installed that's missing in ```000-install-dependencies```. Please reach out if that's the case. The Media SDK is included in the FFmpeg build for Intel hardware, but not yet tested what else is needed for that platform.

Remember to add ```/usr/local/bin``` to your ```PATH``` environment variable if not already done, preferably before ```/usr/bin```.

## x264 and x265 multilibs

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

## Determine the VAAPI driver to use

For hardware acceleration to work, the browser may have the VAAPI driver built-in or you will need a suitable driver, i.e. ```ls /usr/lib64/dri/*_drv_video.so```. To be sure run ```vainfo``` in a terminal window. For AMD, try ```LIBVA_DRIVER_NAME=r600 vainfo``` or ```LIBVA_DRIVER_NAME=radeonsi vainfo```. For Intel, the ```iHD``` driver is newer. So check first ```LIBVA_DRIVER_NAME=iHD vainfo```. Otherwise, try ```LIBVA_DRIVER_NAME=i965 vainfo```. Below see captured output for the ```nvidia``` VAAPI driver.

```bash
$ LIBVA_DRIVER_NAME=nvidia vainfo

libva info: VA-API version 1.11.0
libva info: User environment variable requested driver 'nvidia'
libva info: Trying to open /usr/lib64/dri/nvidia_drv_video.so
libva info: Found init function __vaDriverInit_1_11
libva info: va_openDriver() returns 0
vainfo: VA-API version: 1.11 (libva 2.11.0)
vainfo: Driver version: Splitted-Desktop Systems VDPAU backend for VA-API - 0.7.4
vainfo: Supported profile and entrypoints
      VAProfileMPEG2Simple            : VAEntrypointVLD
      VAProfileMPEG2Main              : VAEntrypointVLD
      VAProfileMPEG4Simple            : VAEntrypointVLD
      VAProfileMPEG4AdvancedSimple    : VAEntrypointVLD
      VAProfileH264ConstrainedBaseline: VAEntrypointVLD
      VAProfileH264Main               : VAEntrypointVLD
      VAProfileH264High               : VAEntrypointVLD
      VAProfileVC1Simple              : VAEntrypointVLD
      VAProfileVC1Main                : VAEntrypointVLD
      VAProfileVC1Advanced            : VAEntrypointVLD
      VAProfileVP9Profile0            : VAEntrypointVLD
```

## Firefox config

The following is my Firefox config. Adjust the value for ```LIBVA_DRIVER_NAME``` accordingly.

```bash
$ cat ~/.config/firefox.conf

export LIBVA_DRIVERS_PATH=/usr/lib64/dri
export LIBVA_DRIVER_NAME=nvidia
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

## Firefox settings

Below are the minimum settings applied via ```about:config``` to enable hardware acceleration. The ```media.rdd-ffmpeg.enable``` flag must be enabled for h264ify to work along with VP9. Basically, this allows you to choose to play videos via the h264ify extension or the VP9 format by disabling h264ify and enjoy beyond 1080P playback.

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

## Chromium installation and configuration

[Chromium](https://dev.chromium.org/Home) is an open-source browser project. Some say it's a browser made for developers. The [chromium-latest-linux](https://github.com/scheib/chromium-latest-linux) repository for launching Chromium works great including VP9 video playback. Unfortunately, the browswer cannot decode H.264-ACC media.

**Edit run.sh**

Insert lines exporting ```LIBVA_DRIVERS_PATH```, ```LIBVA_DRIVER_NAME```, and ```LD_LIBRARY_PATH```. Adjust the value for ```LIBVA_DRIVER_NAME``` accordingly. Insert additional lines to rid of the ```Google API keys are missing``` notification.

Decoding videos using hardware acceleration requires the ```--enable-features=VaapiVideoDecoder``` flag. In addition the ```--use-gl=egl``` or ```--use-gl=desktop``` flag is needed depending on runninng ```wayland``` or ```x11```. See [wiki](https://wiki.archlinux.org/title/Chromium) at Arch Linux for optional flags.

Opening new windows may be larger then the initial window. After a while, that can be annoying. The extra ```--window-size=x,y``` flag resolves that. Adjust the values in pixels to your liking.

```bash
#! /bin/bash

export LIBVA_DRIVERS_PATH=/usr/lib64/dri
export LIBVA_DRIVER_NAME=nvidia
export LD_LIBRARY_PATH=/opt/nvidia/lib64:/usr/local/lib

# To rid of the Google API keys are missing notification.
export GOOGLE_API_KEY=no
export GOOGLE_DEFAULT_CLIENT_ID=no
export GOOGLE_DEFAULT_CLIENT_SECRET=no

BASEDIR=$(dirname $0)

if [ $XDG_SESSION_TYPE == wayland ]
then
    $BASEDIR/latest/chrome --window-size=1100,900 \
        --use-gl=egl --enable-features=VaapiVideoDecoder \
        --user-data-dir="$DATADIR" $* &> /dev/null &
elif [ $XDG_SESSION_TYPE == x11 ]
then
    $BASEDIR/latest/chrome --window-size=1100,900 \
        --use-gl=desktop --enable-features=VaapiVideoDecoder \
        --user-data-dir="$DATADIR" $* &> /dev/null &
fi
```

**First time**

On first launch, go into ```Settings -> Appearance -> Customize fonts``` and change the default fonts to your liking. On Clear Linux, Standard font ```Noto Sans```, Serif font ```Noto Serif```, and Sans-serif font ```Noto Sans``` look great.

```bash
$ ./update-and-run.sh
```

**Subsequently**

```bash
$ ./run.sh
```

## Google Chrome installation and run script

[Google Chrome](https://www.google.com/chrome/) is an open-source browser built by Google. Imaging that! You will find that the browser is quite fast. For NVIDIA hardware, one nicety is that video playback utilizes the Video Engine along with the GPU. That is quite awesome and saves me 15 watts versus Chromium and Firefox.

The ```RPM``` file for Google Chrome can be found at [Google](https://www.google.com/chrome/) and [pkgs.org](https://pkgs.org/download/google-chrome). At the time of writing, I installed version 97.0.4692.71. **Note:** Installing Google Chrome will add the Google repository so your system will automatically keep Google Chrome up to date. If you don't want Google's repository, do ```sudo touch /etc/default/google-chrome``` before installing the package. Not knowing if this is true on Clear Linux, I'm choosing to check manually for updates periodically at pkgs.org.

```bash
$ sudo mkdir -p /etc/default
$ sudo touch /etc/default/google-chrome
# install file from Google (or)
$ sudo rpm -ivh ~/Downloads/google-chrome-stable_current_x86_64.rpm --nodeps
# install file from pkgs.org, change version accordingly
$ sudo rpm -ivh ~/Downloads/google-chrome-stable-97.0.4692.71-1.x86_64.rpm --nodeps
```

**Create run script**

This resembles the run script for Chromium. Be sure to adjust the value for ```LIBVA_DRIVER_NAME```. See [wiki](https://wiki.archlinux.org/title/Google_chrome) at Arch Linux for optional flags.

```bash
#! /bin/bash
#  filename: run-chrome.sh

export LIBVA_DRIVERS_PATH=/usr/lib64/dri
export LIBVA_DRIVER_NAME=nvidia
export LD_LIBRARY_PATH=/opt/nvidia/lib64:/usr/local/lib

if [ $XDG_SESSION_TYPE == wayland ]
then
    google-chrome-stable --window-size=1100,900 --use-gl=egl \
        --enable-features=VaapiVideoDecoder $* &> /dev/null &
elif [ $XDG_SESSION_TYPE == x11 ]
then
    google-chrome-stable --window-size=1100,900 --use-gl=desktop \
        --enable-features=VaapiVideoDecoder $* &> /dev/null &
fi
```

**Make executable**

```bash
$ chmod 755 run-chrome.sh
```

**Run**

On first launch (just like with Chromium), you may want to go into ```Settings -> Appearance -> Customize fonts``` and change the default fonts to your liking. On Clear Linux, Standard font ```Noto Sans```, Serif font ```Noto Serif```, and Sans-serif font ```Noto Sans``` look great.

```
$ ./run-chrome.sh
```

## Vivaldi installation and run script

[Vivaldi](https://vivaldi.com) is yet another open-source browser. The main highlight is being able to communicate in a much more organized way, while keeping control of your data. That sounds delightful! For NVIDIA hardware, one nicety is that video playback also utilizes the Video Engine along with the GPU. This is similarly to Google Chrome.

The ```RPM``` file for Vivaldi can be found at [Vivaldi](https://vivaldi.com/download/). At the time of writing, I installed version 5.0.2497.38. **Note:** Installing Vivaldi will add the Vivaldi repository so your system will automatically keep Vivaldi up to date. If you don't want Vivaldi's repository, do ```sudo touch /etc/default/vivaldi``` before installing the package. Not knowing if this is true on Clear Linux, I'm choosing to check manually for updates periodically at vivaldi.com.

```bash
$ sudo mkdir -p /etc/default
$ sudo touch /etc/default/vivaldi
# install file from Vivaldi, change version accordingly
$ sudo rpm -ivh ~/Downloads/vivaldi-stable-5.0.2497.38-1.x86_64.rpm --nodeps
```

**Create run script**

This resembles closely the Google Chrome run script. Be sure to adjust the value for ```LIBVA_DRIVER_NAME```.

```bash
#! /bin/bash
#  filename: run-vivaldi.sh

export LIBVA_DRIVERS_PATH=/usr/lib64/dri
export LIBVA_DRIVER_NAME=nvidia
export LD_LIBRARY_PATH=/opt/nvidia/lib64:/usr/local/lib

if [ $XDG_SESSION_TYPE == wayland ]
then
    vivaldi-stable --window-size=1100,900 --use-gl=egl \
        --enable-features=VaapiVideoDecoder $* &> /dev/null &
elif [ $XDG_SESSION_TYPE == x11 ]
then
    vivaldi-stable --window-size=1100,900 --use-gl=desktop \
        --enable-features=VaapiVideoDecoder $* &> /dev/null &
fi
```

**Make executable**

```bash
$ chmod 755 run-vivaldi.sh
```

**Run**

On first launch, go into ```Settings -> Webpages -> Fonts``` and change the default fonts to your liking. On Clear Linux, Standard font ```Noto Sans```, Serif font ```Noto Serif```, and Sans-serif font ```Noto Sans``` look great.

```
$ ./run-vivaldi.sh
```

## How can I make sure hardware acceleration is working?

In Chromium, Google Chrome, and Vivaldi, check the ```chrome://gpu``` page. In Firefox, check ```about::support``` page. Another way is running a utility suited for your hardware, while watching a video.

1. ```watch -n 1 /opt/nvidia/bin/nvidia-smi``` to check if "GPU-Util" percentage goes up
2. ```sudo intel_gpu_top``` to check if percentage under the "Video" section goes up
3. ```watch -n 1 sudo intel_gpu_frequency``` to check if the frequency goes up

## HDR videos

To play HDR videos, see ```youtube-play``` found in the extras folder.

## See also, annoucement at Clear Linux plus wikis at Arch Linux

* [Annoucement and benchmarks](https://community.clearlinux.org/t/ffmpeg-supporting-h-264-and-vp9-hardware-acceleration-in-firefox/6148)
* [Hardware video acceleration](https://wiki.archlinux.org/title/Hardware_video_acceleration)
* [Chromium](https://wiki.archlinux.org/title/Chromium)
* [Firefox](https://wiki.archlinux.org/title/Firefox)
* [Google Chrome](https://wiki.archlinux.org/title/Google_chrome)
* [Vivaldi](https://wiki.archlinux.org/title/Vivaldi)

