# 🍎 Plex tvOS Direct Play Profile

An optimized Plex `tvOS.xml` device profile specifically designed for the **Apple TV 4K (2022, A15 Bionic — model AppleTV14,2)**. This profile maximizes Direct Play support and eliminates unnecessary transcoding, delivering the best possible audiovisual quality on the latest Apple TV hardware.

---

## 🎯 What This Does

Plex uses device profiles to decide whether a media file can be **Direct Played** (sent as-is with zero quality loss) or needs to be **transcoded** (re-encoded on-the-fly, consuming server CPU and degrading quality). The stock Plex tvOS profile is deliberately conservative — it often transcodes files that the Apple TV 4K is perfectly capable of playing natively.

This profile unlocks the full hardware capabilities of the A15 Bionic chip, enabling Direct Play for:

- **Video:** HEVC/H.265 up to 12-bit (HDR10, Dolby Vision), AV1, H.264 up to 4K Level 5.1, ProRes, VP9
- **Audio:** AAC, AC3, E-AC3 (Dolby Digital Plus), FLAC, ALAC, Opus, PCM — up to 8 channels
- **Subtitles:** SRT, ASS, SSA, PGS (Blu-ray), VobSub (DVD), WebVTT, mov_text/tx3g
- **Containers:** MKV, MP4, MOV, M4V, MPEG-TS (HLS)

---

## 📱 Target Device

| Specification | Value |
|---|---|
| **Device** | Apple TV 4K (3rd generation, 2022) |
| **Model** | AppleTV14,2 |
| **Chip** | Apple A15 Bionic |
| **GPU** | 5-core Apple GPU |
| **Video Decoders** | H.264, H.265/HEVC (up to 12-bit), AV1, VP9, ProRes |
| **Audio Decoders** | AAC, AC-3, E-AC3, FLAC, ALAC |
| **HDR Support** | HDR10, Dolby Vision, HLG |
| **Max Resolution** | 3840×2160 (4K) at 60fps |
| **tvOS** | 16+ recommended |

---

## 📦 Installation

### Option A: Bind Mount (Recommended — Persistent Across Updates)

1. **On your Plex server**, create a profiles directory:
   ```bash
   mkdir -p /path/to/plex/profiles
   ```

2. **Download the file:**
   ```bash
   curl -o /path/to/plex/profiles/tvOS.xml \
     https://raw.githubusercontent.com/jasimancas/plex-tvos-directplay/main/tvOS.xml
   ```

3. **Add a bind mount** to your `docker-compose.yml` (or `docker run` command):
   ```yaml
   services:
     plex:
       image: lscr.io/linuxserver/plex:latest
       volumes:
         - /path/to/plex/profiles:/usr/lib/plexmediaserver/Resources/Profiles
         # ... your other volumes
   ```

4. **Restart Plex:**
   ```bash
   docker compose up -d plex
   ```

### Option B: Direct Copy (Not Persistent — Lost on Container Recreation)

```bash
docker cp tvOS.xml plex:/usr/lib/plexmediaserver/Resources/Profiles/tvOS.xml
docker restart plex
```

> ⚠️ **Warning:** Option B files are overwritten when the container image is updated. Option A (bind mount) is strongly recommended.

---

## 🔍 Verification

After restarting Plex, verify the profile is active:

```bash
docker exec plex cat /usr/lib/plexmediaserver/Resources/Profiles/tvOS.xml | head -5
```

Then play a 4K HEVC MKV file from your Apple TV — check the Plex Dashboard to confirm it shows **Direct Play** (not transcoding).

---

## 📖 Profile Structure — Section by Section

The `tvOS.xml` file is a Plex device profile that tells the server exactly what the Apple TV can and cannot play. Here's a detailed breakdown of every section:

### `<Settings>`

General behavior settings for the client.

```xml
<Setting name="DirectPlayStreamSelection" value="true" />
<Setting name="StreamUnselectedIncompatibleAudioStreams" value="true" />
```

| Setting | Purpose |
|---|---|
| `DirectPlayStreamSelection` | Lets the client (not the server) pick which stream to Direct Play — faster and more reliable selection |
| `StreamUnselectedIncompatibleAudioStreams` | If an audio stream is incompatible, it's automatically transcoded alongside the Direct Play video, instead of forcing a full video transcode |

---

### `<TranscodeTargets>`

Defines what format Plex transcodes **to** when Direct Play isn't possible. These targets use formats the Apple TV is guaranteed to support.

```xml
<VideoProfile protocol="hls" container="mpegts" codec="h264,hevc,av1"
             audioCodec="aac,eac3,ac3,flac"
             subtitleCodec="eia_608_embedded,mov_text,text" context="streaming">
  <Setting name="HlsExtraMultiChannelAudioStream" value="ac3" />
</VideoProfile>
```

| Attribute | Value | Why |
|---|---|---|
| `protocol` | `hls` | HTTP Live Streaming — Apple's native streaming protocol |
| `container` | `mpegts` | MPEG Transport Stream — optimal for HLS delivery |
| `codec` | `h264,hevc,av1` | All three are hardware-accelerated on the A15 Bionic |
| `audioCodec` | `aac,eac3,ac3,flac` | AAC stereo baseline + multi-channel EAC3/AC3/FLAC |
| `context` | `streaming` | This profile is used for live streaming (not file download) |
| `HlsExtraMultiChannelAudioStream` | `ac3` | Adds a secondary AC3 stream for multi-channel audio — without this, the Apple TV only gets stereo when transcoding |

A `static` (non-streaming) transcode target is also included for direct file downloads:

```xml
<VideoProfile container="mp4" codec="h264" audioCodec="aac,eac3,ac3" context="static" />
```

Subtitle transcoding profiles are included to convert unsupported subtitle formats:

```xml
<SubtitleProfile container="ass" codec="ass" context="all" />
<SubtitleProfile container="ssa" codec="ssa" context="all" />
<SubtitleProfile container="smi" codec="smi" context="all" />
<SubtitleProfile container="webvtt" codec="webvtt" context="all" />
```

---

### `<DirectPlayProfiles>`

The heart of the profile — defines which media combinations can be played **without any transcoding**. This is where most of the magic happens.

#### MKV Container (Primary Library Format)

```xml
<VideoProfile container="mkv"
             codec="hevc,h265,h264,av1,avc1,vp9,mpeg4"
             audioCodec="aac,ac3,eac3,flac,alac,opus,pcm,mp3"
             subtitleCodec="mov_text,tx3g,ttxt,text,srt,ass,ssa,pgssub,vobsub,subrip,webvtt" />
```

| Attribute | Codecs | Why |
|---|---|---|
| `container` | `mkv` | Matroska — the most common container for home media libraries (especially after tools like Tdarr process them) |
| `codec` | `hevc,h265,h264,av1,avc1,vp9,mpeg4` | Full coverage of modern and legacy codecs — HEVC for 4K HDR, AV1 for future-proofing, H.264 for compatibility, VP9 for web content |
| `audioCodec` | `aac,ac3,eac3,flac,alac,opus,pcm,mp3` | From lossless FLAC/ALAC to standard AC3/EAC3 to lossy AAC/MP3 |
| `subtitleCodec` | `mov_text,tx3g,ttxt,text,srt,ass,ssa,pgssub,vobsub,subrip,webvtt` | Every subtitle format the Apple TV can handle — SRT for standard subs, PGS for Blu-ray, VobSub for DVD, ASS/SSA for styled subs, WebVTT for modern |

#### MP4/M4V Container (with ProRes)

```xml
<VideoProfile container="mp4,m4v"
             codec="h264,mpeg4,hevc,h265,av1,avc1,prores"
             audioCodec="aac,ac3,eac3,flac,alac,opus,pcm"
             subtitleCodec="mov_text,tx3g,ttxt,text,srt,ass,pgssub,subrip,webvtt" />
```

Includes **ProRes** — Apple's professional video codec, supported natively by the A15 Bionic. Useful for content created in Final Cut Pro.

#### HTTP Protocol Direct Play (Critical Fix)

```xml
<VideoProfile protocol="http" container="mp4,mkv,mov,m4v"
             codec="h264,hevc,h265,av1,avc1,mpeg4,prores"
             audioCodec="aac,ac3,eac3,flac,alac,opus,pcm"
             subtitleCodec="mov_text,tx3g,ttxt,text,srt,ass,pgssub,subrip,webvtt" />
```

> 🔑 **This is the most important addition.** Without `protocol="http"`, Plex logs the error:
> ```
> No direct play video profile exists for protocol http with container mkv, and video codec hevc
> ```
> This forces transcoding even when the file is perfectly compatible. Adding this line **fixes** that error.

#### HLS Direct Play

```xml
<VideoProfile protocol="hls" container="mpegts"
             codec="h264,hevc,av1" audioCodec="aac,eac3,ac3" />
```

Allows direct playback of HLS streams (like live TV or internet streams) without transcoding.

#### Music Direct Play

```xml
<MusicProfile container="mp3" codec="mp3" />
<MusicProfile container="mp4" codec="aac" />
<MusicProfile container="flac" codec="flac" />
<MusicProfile container="m4a" codec="alac" />
<MusicProfile container="ogg" codec="opus,vorbis" />
```

All major music formats get Direct Play — MP3, AAC, FLAC, ALAC (Apple Lossless), Opus, and Vorbis.

---

### `<TranscodeTargetProfiles>`

When transcoding is unavoidable, this limits what the transcoder outputs — ensuring the transcoded stream is always compatible.

```xml
<VideoTranscodeTarget protocol="*" context="all">
  <VideoCodec name="h264">
    <Limitations>
      <UpperBound name="video.bitDepth" value="8" />
    </Limitations>
  </VideoCodec>
</VideoTranscodeTarget>
```

Transcoded video is always limited to **H.264 8-bit** — the most universally compatible format. This prevents the transcoder from outputting formats the Apple TV can't handle in edge cases.

---

### `<CodecProfiles>`

Defines specific limitations for each codec. These act as filters — if a file exceeds these limits, Plex will transcode instead of Direct Playing.

#### HEVC / H.265

```xml
<VideoCodec name="hevc|h265">
  <Limitations>
    <UpperBound name="video.width" value="3840" isRequired="false" />
    <UpperBound name="video.height" value="2160" isRequired="false" />
    <UpperBound name="video.bitDepth" value="12" />
    <NotMatch name="video.separateFields" value="1" />
  </Limitations>
</VideoCodec>
```

| Limit | Value | Reason |
|---|---|---|
| Width / Height | 3840×2160 (4K) | Maximum resolution the device supports |
| `isRequired="false"` | — | If the file's resolution metadata is missing, don't force a transcode — try Direct Play anyway |
| Bit depth | 12 | Enables HDR10 (10-bit) and Dolby Vision (12-bit) Direct Play |
| `separateFields` | reject 1 | Field-separated content (interlaced) isn't playable |

#### AV1

Same limits as HEVC — up to 4K 12-bit. AV1 is the next-gen codec supported by the A15 Bionic but missing from the stock profile.

#### H.264 / AVC

```xml
<VideoCodec name="h264|avc1">
  <Limitations>
    <LowerBound name="video.width" value="64" isRequired="false" />
    <LowerBound name="video.height" value="64" isRequired="false" />
    <UpperBound name="video.width" value="3840" isRequired="false" />
    <UpperBound name="video.height" value="2160" isRequired="false" />
    <UpperBound name="video.bitDepth" value="8" />
    <UpperBound name="video.level" value="51" />
    <UpperBound name="video.refFrames" value="16" />
    <Match name="video.profile" list="baseline|main|high" />
    <NotMatch name="video.separateFields" value="1" />
  </Limitations>
</VideoCodec>
```

| Limit | Value | Reason |
|---|---|---|
| Resolution | 64px to 3840×2160 | Full range from tiny thumbnails to 4K |
| Bit depth | 8 | H.264 on Apple TV is 8-bit only |
| Level | 5.1 | Level 5.1 enables 4K H.264 — the stock profile omits this, forcing transcoding for 4K H.264 content |
| refFrames | 16 | Reference frames limit — files with too many ref frames can overwhelm the decoder |
| Profile | baseline, main, high | Only these H.264 profiles are supported |
| `separateFields` | reject 1 | No interlaced content |

#### VP9

Up to 4K 10-bit. Primarily used for web video (YouTube, etc.) — included for completeness.

#### ProRes

Up to 4K 12-bit. Apple's professional codec, included natively since it's hardware-accelerated.

#### Audio Codec Limits

```xml
<VideoAudioCodec name="aac">
  <Limitations>
    <UpperBound name="audio.channels" value="2" onlyTranscodes="true" />
  </Limitations>
</VideoAudioCodec>

<VideoAudioCodec name="ac3|eac3|flac|alac">
  <Limitations>
    <UpperBound name="audio.channels" value="8" />
    <Match name="audio.samplingRate" list="44100|48000|88200|96000|176400|192000" />
  </Limitations>
</VideoAudioCodec>

<VideoAudioCodec name="opus|pcm|mp3">
  <Limitations>
    <UpperBound name="audio.channels" value="8" />
  </Limitations>
</VideoAudioCodec>
```

| Codec | Channel Limit | Notes |
|---|---|---|
| AAC | 2.0 (transcode only) | `onlyTranscodes="true"` means AAC multi-channel is Direct Played if present, but if transcoding is needed, it's limited to stereo |
| AC3 / E-AC3 | 8 channels | Dolby Digital / Dolby Digital Plus — standard for surround sound |
| FLAC / ALAC | 8 channels | Lossless audio — ALAC is Apple's lossless format |
| Opus / PCM / MP3 | 8 channels | Opus for web audio, PCM for uncompressed, MP3 for legacy |
| Sampling rate | 44.1 – 192 kHz | All standard rates for AC3/EAC3/FLAC/ALAC |

---

## 🆚 Comparison: Stock vs. This Profile

| Feature | Stock Plex Profile | This Profile |
|---|---|---|
| HEVC bit depth | 10-bit | **12-bit** (Dolby Vision) |
| AV1 | ❌ | ✅ |
| ProRes | ❌ | ✅ |
| VP9 | ❌ | ✅ |
| MKV container | ❌ | ✅ |
| HTTP protocol Direct Play | ❌ | ✅ (fixes HEVC/MKV transcoding bug) |
| HLS Direct Play | Limited | ✅ (H.264 + HEVC + AV1) |
| PGS subtitles (Blu-ray) | ❌ | ✅ |
| VobSub subtitles (DVD) | ❌ | ✅ |
| WebVTT subtitles | ❌ | ✅ |
| ASS/SSA subtitles | ❌ | ✅ |
| FLAC audio | ✅ (limited) | ✅ (8 channels) |
| ALAC audio | ✅ (limited) | ✅ (8 channels) |
| Opus audio | ❌ | ✅ |
| H.264 Level 5.1 (4K) | ❌ | ✅ |
| Per-codec channel limits | ❌ | ✅ |
| `isRequired="false"` | ❌ | ✅ (avoids transcoding when metadata is missing) |
| HLS multi-channel audio | ❌ | ✅ (`HlsExtraMultiChannelAudioStream`) |
| TranscodeTargetProfiles | ❌ | ✅ (ensures compatible transcoded output) |
| Subtitle transcoding | ❌ | ✅ (ASS/SSA/SMI/WebVTT) |

---

## 🏗️ Methodology

This profile was built by auditing all **78 stock Plex device profiles** (including iOS, Xbox One, Chromecast, Samsung Tizen, Android, PlayStation 4, and Plex Desktop) and extracting the best practices from each, combined with Apple's official specifications for the Apple TV 4K (2022).

---

## 📄 License

MIT — do whatever you want with it. Attribution appreciated but not required.

---

## ⚠️ Disclaimer

This profile is designed for the **Apple TV 4K (2022, A15 Bionic)**. It may work on earlier Apple TV models (4K 2021, HD) but hasn't been tested on them. If you have an older model, some codecs (like AV1) may not be supported and could cause playback issues.


