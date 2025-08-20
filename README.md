
````markdown
# 🎥 Mathcast Bulk Downloader

Bulk-download **ASU Mathcast (Opencast)** lecture videos using a PowerShell script and [yt-dlp](https://github.com/yt-dlp/yt-dlp).  
No manual clicking through hundreds of pages — the script talks to the Opencast API, collects signed stream URLs (.m3u8 / .mp4), and feeds them to yt-dlp for reliable, resumable downloads.

---

## ✨ Features
- Reads `links.txt` containing your Mathcast video page URLs  
- Extracts signed HLS / MP4 streams via the Opencast API  
- Generates a `streams.txt` with direct media links  
- Feeds them to yt-dlp for reliable, resumable downloads  
- Skips already downloaded items on re-runs  

---

## 🛠 Requirements (Windows 10/11)

1) **PowerShell 7+**  
   - Check:  
     ```powershell
     $PSVersionTable.PSVersion
     ```

2) **yt-dlp**  
   - Install (any one method):
     ```powershell
     pip install -U yt-dlp
     ```
     or download the Windows EXE from the yt-dlp releases page and place it on PATH.

   - Verify:
     ```powershell
     yt-dlp --version
     ```

3) **FFmpeg** (for HLS merging → MP4)  
   - Recommended (Essentials build):
     ```powershell
     winget install --id=Gyan.FFmpeg.Essentials -e
     ```
   - Verify:
     ```powershell
     ffmpeg -version
     ```

4) **cookies.txt** (browser export while logged into `mathcast.la.asu.edu`)  
   - Use a browser extension like **Get cookies.txt**.  
   - Visit a playable Mathcast video page (already logged in), export cookies for the **mathcast.la.asu.edu** domain into a file named **`cookies.txt`**.  
   - Save `cookies.txt` in the same folder as the script(s).  
   - ⚠️ **Do not commit this file** to your repo.

---

## 📂 Project layout

```text
mathcast-downloader/
├── links.txt              # one Mathcast "Theodul" URL per line (has ?id=...)
├── cookies.txt            # exported from your browser (NOT committed)
├── mathcast.ps1           # one-file script (extract + download)
├── extract_streams.ps1    # optional: split-step variant (extract only)
├── download_streams.ps1   # optional: split-step variant (download only)
├── streams.txt            # auto-generated list of media URLs
└── README.md
````

**Example `links.txt`:**

```text
https://mathcast.la.asu.edu/engage/theodul/ui/core.html?id=aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee
https://mathcast.la.asu.edu/engage/theodul/ui/core.html?id=ffffffff-1111-2222-3333-444444444444
```

---

## 📜 One-file script — `mathcast.ps1`

```powershell
# 🎥 Mathcast Bulk Downloader (one-file)
# Requirements: PowerShell 7+, yt-dlp, ffmpeg, cookies.txt
# Input files:  links.txt   (Theodul URLs with ?id=...)
#               cookies.txt (export from browser while logged into mathcast.la.asu.edu)
# Output:       streams.txt and downloaded videos under .\Mathcast\

$ErrorActionPreference = "Stop"
Set-StrictMode -Version Latest
cd $PSScriptRoot

# ------------------------------
# Configuration
# ------------------------------
$LinksFile          = ".\links.txt"
$CookiesFile        = ".\cookies.txt"
$StreamsFile        = ".\streams.txt"
$DownloadArchive    = ".\downloaded.txt"
$OutputTemplate     = "Mathcast/%(title)s.%(ext)s"
$ConcurrentVideos   = 4
$ConcurrentFragments= 10
$MergeFormat        = "mp4"
$DownloadAllTracks  = $false   # true => download all tracks per event

# ------------------------------
# Sanity checks
# ------------------------------
if (-not (Test-Path $LinksFile))   { Write-Error "Missing $LinksFile"; exit 1 }
if (-not (Test-Path $CookiesFile)) { Write-Error "Missing $CookiesFile"; exit 1 }

# ------------------------------
# Helpers
# ------------------------------
function Get-EventId {
  param([string]$Url)
  if ($Url -match '(?i)[?&]id=([^&]+)') { return $Matches[1] }
  return $null
}

# ------------------------------
# Extract media URLs via Opencast APIs
# ------------------------------
$links   = Get-Content $LinksFile | Where-Object { $_ -match 'id=' }
$streams = New-Object System.Collections.Generic.List[string]

foreach ($u in $links) {
  try {
    $id = Get-EventId $u
    if (-not $id) { Write-Host "Skip (no id): $u"; continue }

    $urls = New-Object System.Collections.Generic.List[string]

    # API v1: External API (signed URLs)
    $api1  = "https://mathcast.la.asu.edu/api/events/$id?sign=true"
    $json1 = (& curl.exe -s -b $CookiesFile $api1)
    if ($json1) {
      try {
        $ev = $json1 | ConvertFrom-Json
        if ($ev -and $ev.publications) {
          foreach ($pub in $ev.publications) {
            $items = @()
            if ($pub.PSObject.Properties.Name -contains 'tracks') { $items += $pub.tracks }
            if ($pub.PSObject.Properties.Name -contains 'media')  { $items += $pub.media  }
            foreach ($t in $items) {
              $mt  = $t.mimetype
              $url = $t.url
              if ($null -ne $url) {
                if ($mt -like 'application/x-mpegURL' -or $url -match '\.m3u8(\?|$)') { $urls.Add($url) }
                elseif ($mt -like 'video/mp4' -or $url -match '\.mp4(\?|$)')         { $urls.Add($url) }
              }
            }
          }
        }
      } catch { }
    }

    # API v2: Legacy search API (fallback)
    if ($urls.Count -eq 0) {
      $api2  = "https://mathcast.la.asu.edu/search/episode.json?id=$id"
      $json2 = (& curl.exe -s -b $CookiesFile $api2)
      if ($json2) {
        try {
          $root = $json2 | ConvertFrom-Json
          $res  = $root.'search-results'.result
          if ($res -and -not ($res -is [System.Array])) { $res = @($res) }
          foreach ($r in $res) {
            $tracks = $r.mediapackage.media.track
            if ($tracks -and -not ($tracks -is [System.Array])) { $tracks = @($tracks) }
            foreach ($t in $tracks) {
              $mt  = $t.mimetype
              $url = $t.url
              if ($null -ne $url) {
                if ($mt -like 'application/x-mpegURL' -or $url -match '\.m3u8(\?|$)') { $urls.Add($url) }
                elseif ($mt -like 'video/mp4' -or $url -match '\.mp4(\?|$)')         { $urls.Add($url) }
              }
            }
          }
        } catch { }
      }
    }

    if ($urls.Count -gt 0) {
      if ($DownloadAllTracks) {
        foreach ($s in $urls) { $streams.Add($s) }
        Write-Host "OK (all tracks): $id"
      } else {
        $streams.Add($urls[0])
        Write-Host "OK: $id"
      }
    } else {
      Write-Host "No streams found for $id"
    }
  } catch {
    Write-Host "Error on $u : $_"
  }
}

$streams | Set-Content $StreamsFile -Encoding ASCII
Write-Host "✅ Wrote $(($streams).Count) stream URLs to $StreamsFile"

# ------------------------------
# Download via yt-dlp
# ------------------------------
$yt = (Get-Command yt-dlp -ErrorAction SilentlyContinue)
if ($yt) {
  & yt-dlp -a $StreamsFile --cookies $CookiesFile `
    -N $ConcurrentVideos --concurrent-fragments $ConcurrentFragments `
    --download-archive $DownloadArchive `
    --merge-output-format $MergeFormat `
    -o $OutputTemplate
} else {
  & python -m yt_dlp -a $StreamsFile --cookies $CookiesFile `
    -N $ConcurrentVideos --concurrent-fragments $ConcurrentFragments `
    --download-archive $DownloadArchive `
    --merge-output-format $MergeFormat `
    -o $OutputTemplate
}

Write-Host "🎉 Done."
```

---

## ▶️ Run the one-file script

From the folder containing the files:

```powershell
# One-time allow local scripts
Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned

# Run the downloader
.\mathcast.ps1
```

All videos will appear in the `Mathcast\` folder. Re-runs safely skip already downloaded items.

---

## 🔒 Git Ignore (for your repo)

Create a `.gitignore` file to keep private and generated files out of Git:

```gitignore
# Authentication
cookies.txt

# Generated
streams.txt
downloaded.txt

# Downloads
Mathcast/
```

---

## 🧩 Troubleshooting

* **Unsupported URL** → You must run the script first (to build `streams.txt`), not feed Theodul page URLs directly to yt-dlp.
* **Could not copy Chrome cookie database** → Close Chrome completely or use `cookies.txt` (recommended).
* **No streams found** → Export cookies again while on a video page; ensure `mathcast.la.asu.edu` cookies are present.
* **ffmpeg not found** → Ensure `ffmpeg -version` works in PowerShell; reinstall if needed.

---

## ✅ Quick Start (TL;DR)

```powershell
# Install tools
pip install -U yt-dlp
winget install --id=Gyan.FFmpeg.Essentials -e

# Export cookies.txt from your browser
# Create links.txt with your Theodul URLs

# Run downloader
.\mathcast.ps1
```

---

## 📄 License & Notes

* Use responsibly and only for content you’re authorized to download.
* This repo intentionally excludes `cookies.txt` and other private info.
* Licensed under MIT (or your choice).

```
 
Do you want me to also generate a **ready-to-use repo ZIP** (with this `README.md`, scripts, and a blank `.gitignore`), so you can push straight to GitHub?
```
