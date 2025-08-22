````markdown
# 🎥 Mathcast Bulk Downloader

Bulk-download ASU Mathcast (Opencast) lecture videos using PowerShell and [yt-dlp](https://github.com/yt-dlp/yt-dlp).

---

## ✨ Features
- Reads `links.txt` with Mathcast video page URLs  
- Extracts signed HLS / MP4 streams via the Opencast API  
- Generates `streams.txt` with direct media links  
- Feeds them to yt-dlp for downloads  
- Skips already downloaded items  

---

## 🛠 Requirements

```powershell
# Check PowerShell version
$PSVersionTable.PSVersion

# Install yt-dlp
pip install -U yt-dlp

# Install FFmpeg
winget install --id=Gyan.FFmpeg.Essentials -e

# Verify installs
yt-dlp --version
ffmpeg -version
````

Export `cookies.txt` from your browser while logged into `mathcast.la.asu.edu`. Place it with the scripts.

---

## 📂 Project layout

```text
mathcast-downloader/
├── links.txt
├── cookies.txt
├── mathcast.ps1
├── extract_streams.ps1
├── download_streams.ps1
├── streams.txt
└── README.md
```

Example `links.txt`:

```text
https://mathcast.la.asu.edu/engage/theodul/ui/core.html?id=aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee
https://mathcast.la.asu.edu/engage/theodul/ui/core.html?id=ffffffff-1111-2222-3333-444444444444
```

---

## 📜 One-file script — `mathcast.ps1`

```powershell
$ErrorActionPreference = "Stop"
Set-StrictMode -Version Latest
cd $PSScriptRoot
$LinksFile          = ".\links.txt"
$CookiesFile        = ".\cookies.txt"
$StreamsFile        = ".\streams.txt"
$DownloadArchive    = ".\downloaded.txt"
$OutputTemplate     = "Mathcast/%(title)s.%(ext)s"
$ConcurrentVideos   = 4
$ConcurrentFragments= 10
$MergeFormat        = "mp4"
$DownloadAllTracks  = $false
if (-not (Test-Path $LinksFile))   { Write-Error "Missing $LinksFile"; exit 1 }
if (-not (Test-Path $CookiesFile)) { Write-Error "Missing $CookiesFile"; exit 1 }
function Get-EventId { param([string]$Url) if ($Url -match '(?i)[?&]id=([^&]+)') { return $Matches[1] } return $null }
$links   = Get-Content $LinksFile | Where-Object { $_ -match 'id=' }
$streams = New-Object System.Collections.Generic.List[string]
foreach ($u in $links) {
  try {
    $id = Get-EventId $u
    if (-not $id) { Write-Host "Skip (no id): $u"; continue }
    $urls = New-Object System.Collections.Generic.List[string]
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
      if ($DownloadAllTracks) { foreach ($s in $urls) { $streams.Add($s) } }
      else { $streams.Add($urls[0]) }
      Write-Host "OK: $id"
    } else {
      Write-Host "No streams found for $id"
    }
  } catch {
    Write-Host "Error on $u : $_"
  }
}
$streams | Set-Content $StreamsFile -Encoding ASCII
Write-Host "✅ Wrote $(($streams).Count) stream URLs to $StreamsFile"
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
```

---

## ▶️ Run the one-file script

```powershell
Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned
.\mathcast.ps1
```

---

## ✂️ Two-file variant

**`extract_streams.ps1`**

```powershell
$ErrorActionPreference = "Stop"
Set-StrictMode -Version Latest
cd $PSScriptRoot
$LinksFile   = ".\links.txt"
$CookiesFile = ".\cookies.txt"
$StreamsFile = ".\streams.txt"
$DownloadAllTracks = $false
if (-not (Test-Path $LinksFile))   { Write-Error "Missing $LinksFile"; exit 1 }
if (-not (Test-Path $CookiesFile)) { Write-Error "Missing $CookiesFile"; exit 1 }
function Get-EventId([string]$Url) { if ($Url -match '(?i)[?&]id=([^&]+)') { return $Matches[1] } return $null }
$links   = Get-Content $LinksFile | Where-Object { $_ -match 'id=' }
$streams = New-Object System.Collections.Generic.List[string]
foreach ($u in $links) {
  try {
    $id = Get-EventId $u
    if (-not $id) { continue }
    $urls = New-Object System.Collections.Generic.List[string]
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
              $mt=$t.mimetype; $url=$t.url
              if ($url) {
                if ($mt -like 'application/x-mpegURL' -or $url -match '\.m3u8(\?|$)') { $urls.Add($url) }
                elseif ($mt -like 'video/mp4' -or $url -match '\.mp4(\?|$)')        { $urls.Add($url) }
              }
            }
          }
        }
      } catch { }
    }
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
              $mt=$t.mimetype; $url=$t.url
              if ($url) {
                if ($mt -like 'application/x-mpegURL' -or $url -match '\.m3u8(\?|$)') { $urls.Add($url) }
                elseif ($mt -like 'video/mp4' -or $url -match '\.mp4(\?|$)')        { $urls.Add($url) }
              }
            }
          }
        } catch { }
      }
    }
    if ($urls.Count -gt 0) {
      if ($DownloadAllTracks) { foreach ($s in $urls) { $streams.Add($s) } }
      else { $streams.Add($urls[0]) }
      Write-Host "OK: $id"
    } else {
      Write-Host "No streams for $id"
    }
  } catch {
    Write-Host "Error on $u : $_"
  }
}
$streams | Set-Content $StreamsFile -Encoding ASCII
```

**`download_streams.ps1`**

```powershell
$ErrorActionPreference = "Stop"
Set-StrictMode -Version Latest
cd $PSScriptRoot
$StreamsFile       = ".\streams.txt"
$CookiesFile       = ".\cookies.txt"
$DownloadArchive   = ".\downloaded.txt"
$OutputTemplate    = "Mathcast/%(title)s.%(ext)s"
$ConcurrentVideos  = 4
$ConcurrentFragments = 10
$MergeFormat       = "mp4"
if (-not (Test-Path $StreamsFile)) { Write-Error "Missing $StreamsFile"; exit 1 }
if (-not (Test-Path $CookiesFile)) { Write-Error "Missing $CookiesFile"; exit 1 }
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
```

---

## ▶️ Run the two-file scripts

```powershell
.\extract_streams.ps1
.\download_streams.ps1
```

---

## 🔒 .gitignore

```gitignore
cookies.txt
streams.txt
downloaded.txt
Mathcast/
```

---

## ✅ Quick Start

```powershell
pip install -U yt-dlp
winget install --id=Gyan.FFmpeg.Essentials -e
Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned
.\mathcast.ps1
```

