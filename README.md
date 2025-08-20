# opencast-downloader
Batch downloader for ASU Mathcast (Opencast) videos using PowerShell + yt-dlp
# Mathcast Downloader

This PowerShell script downloads ASU Mathcast (Opencast) videos in bulk using the official API and [yt-dlp](https://github.com/yt-dlp/yt-dlp).

## Requirements
- Windows with PowerShell 7+
- [yt-dlp](https://github.com/yt-dlp/yt-dlp) installed (`pip install -U yt-dlp` or standalone EXE)
- [FFmpeg](https://ffmpeg.org/) installed (via winget or manual)
- A valid `cookies.txt` file exported from Chrome/Firefox while logged into Mathcast

## Usage
1. Put your Mathcast links in `links.txt` (one per line, e.g. https://mathcast.la.asu.edu/engage/theodul/ui/core.html?id=...).
2. Export cookies for `mathcast.la.asu.edu` and save as `cookies.txt`.
3. Run the script in PowerShell:

```powershell
.\mathcast.ps1

It will generate streams.txt and then call yt-dlp to download all videos into the Mathcast folder.
