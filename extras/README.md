# youtube-play
A YouTube complementary player for YouTube Downloader.

The script was created to test FFmpeg built with ```nvdec``` and ```nvenc``` hardware acceleration. One use-case is being able to watch HDR videos from YouTube.

## Requirements
Install YouTube Downloader.
```bash
sudo -H pip install --upgrade youtube-dl
```

## Usage
The YouTube URL is Best 8k HDR of 2020 Dolby Vision.
```bash
youtube-play URL -F, --list-formats
youtube-play https://youtu.be/Jz9TdfXlTgs -F
youtube-play https://youtu.be/Jz9TdfXlTgs --list-formats

youtube-play URL [VIDEO_FORMAT_CODE]
youtube-play https://youtu.be/Jz9TdfXlTgs
youtube-play https://youtu.be/Jz9TdfXlTgs 136
```