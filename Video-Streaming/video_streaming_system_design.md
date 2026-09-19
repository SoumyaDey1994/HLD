# Video Streaming Pipeline — Compact End-to-End Flow

**Assumption:** The platform already has the raw **2K MP4** video in S3/Object Storage.

## Overall Architecture

```text
                    BACKEND / PROCESSING PIPELINE

Raw 2K MP4
    │
    ▼
S3 / Object Storage
    │
    ▼
Media Service
    │
    ▼
Virus Scan + Moderation + Media Probe
    │
    ▼
Kafka → Transcoding / Encoding Workers
    │
    ▼
Multiple Renditions
2K / 1080p / 720p / 480p / 360p
    │
    ▼
Chunking + HLS/DASH Packaging
    │
    ▼
Per-Rendition Manifests + Master Manifest
    │
    ▼
S3 / Object Storage
    │
    ▼
Metadata DB → READY
    │
    ▼
Video Available for Consumption


                    PLAYBACK / DELIVERY PIPELINE

User
 │
 ▼
Video API
 │
 ▼
Metadata DB
 │
 ▼
Master Manifest URL
 │
 ▼
CDN
 │
 ▼
Video Player
 │
 ├── Selects initial quality
 │
 ├── Requests segments
 │
 └── Continuously performs ABR
          │
          ▼
   2K / 1080p / 720p / 480p / 360p
```

---

## 1. Raw Video Available in Object Storage

The raw 2K MP4 already exists in S3:

```text
s3://production-media/raw-media/video-123/source.mp4
```

S3 stores the **actual video bytes**.

The Metadata DB stores information such as:

```text
video_id        = video-123
source_key      = raw-media/video-123/source.mp4
resolution      = 2560x1440
format          = mp4
status          = UPLOADED
```

---

## 2. Virus Scanning + Content Moderation + Media Probing

The **Media Service** initiates processing.

The uploaded file goes through:

```text
Virus Scan
    ↓
Content Moderation
    ↓
Media Probe / Validation
```

### Virus Scan

Checks for malware or malicious content.

### Content Moderation

Depending on the platform, checks for prohibited/inappropriate content.

### Media Probe

Extracts/validates technical information:

```text
Resolution: 2560 × 1440
Duration: 20 min
Video Codec: H.264
Audio Codec: AAC
FPS: 30
Bitrate: ...
```

If validation fails:

```text
status = REJECTED
```

If everything passes:

```text
status = PROCESSING
```

---

## 3. Kafka — Asynchronous Processing Pipeline

Once the video is validated, Media Service publishes a processing event:

```text
VideoProcessingRequested
{
    videoId: "video-123",
    sourceKey: "raw-media/video-123/source.mp4"
}
```

to Kafka.

Kafka decouples the API/Media Service from the expensive processing workload.

```text
Media Service
      │
      ▼
    Kafka
      │
      ▼
Transcoding Workers
```

Workers can independently consume jobs and process videos at scale.

---

## 4. Transcoding + Video/Audio Encoding

The original 2K video is converted into multiple **video renditions**.

| Rendition | Resolution | Approx. Bitrate |
|---|---:|---:|
| 2K | 2560×1440 | 8 Mbps |
| 1080p | 1920×1080 | 5 Mbps |
| 720p | 1280×720 | 2.5 Mbps |
| 480p | 854×480 | 1 Mbps |
| 360p | 640×360 | 600 Kbps |

So:

```text
2K Source
   │
   ├── 2K
   ├── 1080p
   ├── 720p
   ├── 480p
   └── 360p
```

Each rendition is encoded using codecs such as:

```text
H.264 / H.265 / AV1
```

Audio is also encoded separately, e.g. AAC/Opus, and later packaged alongside the video.

> **Transcoding creates different resolutions/bitrates; encoding compresses those streams using a codec.**

---

## 5. Chunking / Segmentation

A complete video file is not normally delivered as one large MP4.

Each encoded rendition is divided into small media segments.

For example:

```text
720p
 ├── segment_001.m4s
 ├── segment_002.m4s
 ├── segment_003.m4s
 └── ...
```

Typically, each segment represents a few seconds of playback.

This allows the player to independently request the next small portion of the video.

---

## 6. HLS/DASH Packaging + Manifest Preparation

The encoded segments are packaged into a streaming format such as:

```text
HLS
DASH
```

For HLS, there are effectively **two levels of manifests**.

### Per-Rendition Manifest

For example:

```text
720p/playlist.m3u8
```

It describes:

```text
720p
 ├── segment_001
 ├── segment_002
 ├── segment_003
 └── ...
```

Similarly:

```text
1080p/playlist.m3u8
2K/playlist.m3u8
480p/playlist.m3u8
360p/playlist.m3u8
```

### Master Manifest

Then a master manifest:

```text
master.m3u8
```

references all available renditions:

```text
master.m3u8
 ├── 2K playlist
 ├── 1080p playlist
 ├── 720p playlist
 ├── 480p playlist
 └── 360p playlist
```

The master manifest essentially tells the player:

> These are the different qualities available for this video.

---

## 7. Store Processed Assets in S3

All generated streaming assets are stored in object storage.

For example:

```text
production-media/
└── streaming/
    └── video-123/
        ├── master.m3u8
        │
        ├── 2k/
        │   ├── playlist.m3u8
        │   ├── segment_001.m4s
        │   └── ...
        │
        ├── 1080p/
        │   ├── playlist.m3u8
        │   └── ...
        │
        ├── 720p/
        │   ├── playlist.m3u8
        │   └── ...
        │
        └── 480p/
            ├── playlist.m3u8
            └── ...
```

So S3 now contains both:

```text
Raw source
+
Streaming-ready assets
```

---

## 8. Metadata DB Update + Video Becomes READY

Once all required renditions, segments and manifests are successfully generated:

```text
Media Service
     │
     ▼
Metadata DB
     │
     ▼
status = READY
```

Example:

```text
video_id = video-123
status   = READY
manifest = streaming/video-123/master.m3u8
```

At this point, the video becomes **available for consumption**.

A completion event can also be published:

```text
VideoProcessingCompleted
```

through Kafka so other services can react asynchronously.

---

## 9. Player Requests Video + Selects Initial Quality

When the user clicks **Play**:

```text
User
 ↓
Video API
 ↓
Metadata DB
 ↓
Master Manifest URL
```

The player requests:

```text
master.m3u8
```

through the CDN.

The master manifest tells the player that:

```text
2K
1080p
720p
480p
360p
```

are available.

The player then selects an **initial rendition** based on factors such as:

- Network bandwidth
- Device capability
- Screen resolution
- Current buffer

For example:

```text
Estimated bandwidth = 3 Mbps

             ↓

Select 720p @ 2.5 Mbps
```

---

## 10. CDN + Adaptive Bitrate Streaming

The player starts requesting media segments:

```text
720p/segment_001.m4s
720p/segment_002.m4s
720p/segment_003.m4s
```

The CDN handles delivery.

### CDN Cache Hit

```text
Player
  ↓
CDN
  ↓
Cache HIT
  ↓
Player
```

### CDN Cache Miss

```text
Player
  ↓
CDN
  ↓
Cache MISS
  ↓
S3
  ↓
CDN
  ↓
Player
```

The CDN subsequently caches the segment for future users.

### Adaptive Bitrate

The player continuously monitors:

```text
Bandwidth
Buffer health
Download speed
Device capability
```

Suppose the network deteriorates:

```text
2K
 ↓
1080p
 ↓
720p
```

If the network improves:

```text
720p
 ↓
1080p
 ↓
2K
```

The player simply starts requesting segments from another rendition.

> **ABR means the backend prepares multiple renditions and the player dynamically chooses which rendition's segments to download.**

---

# Final End-to-End Flow

## A. Backend / Video Processing Flow

```text
Raw 2K MP4 already in S3
          │
          ▼
     Media Service
          │
          ▼
Virus Scan + Content Moderation
          │
          ▼
   Media Probe / Validation
          │
          ▼
   status = PROCESSING
          │
          ▼
        Kafka
          │
          ▼
Transcoding + Encoding Workers
          │
          ├── 2K
          ├── 1080p
          ├── 720p
          ├── 480p
          └── 360p
          │
          ▼
     Chunk / Segment
          │
          ▼
    HLS / DASH Packaging
          │
          ├── Per-rendition manifests
          └── Master manifest
          │
          ▼
      Store in S3
          │
          ▼
   Update Metadata DB
          │
          ▼
     status = READY
          │
          ▼
Video becomes available for playback
```

## B. Player / Video Delivery Flow

```text
User clicks PLAY
       │
       ▼
   Video API
       │
       ▼
 Metadata DB
       │
       ▼
Master Manifest URL
       │
       ▼
      CDN
       │
       ▼
    Player
       │
       ▼
Select initial rendition
       │
       ▼
Request media segments
       │
       ├── CDN HIT ──────────► Player
       │
       └── CDN MISS → S3 → CDN → Player
                              │
                              ▼
                    Continuous segment playback
                              │
                              ▼
                     ABR continuously evaluates
                              │
                    ┌─────────┴─────────┐
                    ▼                   ▼
                Network ↓           Network ↑
                    │                   │
                    ▼                   ▼
             Lower rendition       Higher rendition
```

---

## Component Responsibilities

| Component | Responsibility |
|---|---|
| **S3 / Object Storage** | Stores raw video, encoded renditions, manifests and media segments |
| **Media Service** | Orchestrates video processing lifecycle |
| **Virus Scanner** | Malware/security validation |
| **Content Moderation** | Checks content against platform policies |
| **Media Probe** | Extracts and validates technical media metadata |
| **Kafka** | Asynchronous processing jobs and completion events |
| **Transcoding/Encoding Workers** | Create and encode multiple video renditions |
| **Packaging/Segmenter** | Create HLS/DASH manifests and media chunks |
| **Metadata DB** | Stores video state, rendition metadata and manifest locations |
| **CDN** | Caches and delivers manifests/segments close to users |
| **Video Player** | Requests segments, maintains buffer and performs ABR |

### Core Mental Model

```text
S3
 ↓
Media Service
 ↓
Scan + Validate
 ↓
Kafka
 ↓
Transcode + Encode
 ↓
Chunk + Package
 ↓
Manifest + Segments
 ↓
S3
 ↓
Metadata DB = READY
 ↓
CDN
 ↓
Player
 ↓
ABR → Select appropriate rendition
```

> **S3 stores the media, Media Service orchestrates processing, Kafka distributes asynchronous jobs/events, transcoding creates multiple renditions, packaging creates manifests/chunks, Metadata DB tracks lifecycle, and CDN + Player deliver those chunks with ABR dynamically selecting the appropriate quality.**
