# TalkingSanta(5)(2).mp4 Codec Anomaly

A technical forensic breakdown of an anomalous 2011 mobile video export (`TalkingSanta(5)(2).mp4`) that triggers playback bugs, renderer deadlocks, and visual corruption across modern media players (VLC, Discord/Chromium, mobile decoders).

---

## 1. Hashes & File Identification

* **Filename:** `TalkingSanta(5)(2).mp4`
* **File Size:** `235,864` bytes (230.3 KiB)
* **MD5:** `44aacce0673d54fa5f9ecf661bd7a18e`
* **SHA-1:** `011a0370658b823e33ea60ff74f3680b01bb9a84`
* **SHA-256:** `2cc60eac218cc647813cfa0c79fdede79e5cf38844b520c61eef4ddcad34a510`
* **Origin:** *Talking Santa* iOS/Android app (Outfit7, ~2011 release), recorded and exported directly via mobile app.
* **VirusTotal / Public Database matches:** 0 matches (unique local user recording).

---

## 2. Observed Symptoms

1. **Initial Black Screen in VLC (Linux/Windows):**
   * On first launch, VLC renders a black screen for all 6.2 seconds while audio plays. Subsequent launches or software-decoded runs render video.
2. **Discord / Chromium Player Deadlock:**
   * Sending or embedding this video in Discord causes the inline HTML5 player to hang or stop playing until the channel is reloaded or Discord is restarted.
3. **Severe Visual Artifacts on Seeking (Scrubbing):**
   * Attempting to drag the seek bar or scrub forward/backward causes total pixel smear, gray screens, or frozen frames.
4. **End-of-Stream Desync:**
   * Many players freeze on the final frame instead of triggering standard `ended` loop/completion events.

---

## 3. Technical Specifications

### Container: ISO Base Media File Format (`isom / iso2 / mp41`)
* **Box Structure:**
  * `[ftyp]` offset `0`, size `28`
  * `[free]` offset `28`, size `8`
  * `[mdat]` offset `36`, size `233392` (233,384 bytes payload)
  * `[moov]` offset `233428`, size `2436`
* **Creation Timestamp:** `1970-01-01T00:00:00.000000Z` (Unix Epoch 0).
* **Encoder String:** `Lavf53.18.0` (libavformat / FFmpeg 0.8.x era, ~October 2011).

### Video Stream
* **Codec:** MPEG-4 Part 2 Visual (Simple Profile / FourCC: `mp4v` / `0x7634706D`)
* **Resolution:** 320×480 (non-Retina iPhone display native portrait)
* **Framerate:** 10.0 fps fixed (`r_frame_rate=10/1`, `time_base=1/10`)
* **Frame Count:** 62 frames
* **Duration:** 6.200 seconds
* **Bitrate:** ~284 kbps
* **Pixel Format:** `yuv420p`
* **Keyframe (I-frame) Table (`stss`):**
  * `entry_count`: **1**
  * `samples`: **[1]** (ONLY the first frame is a keyframe; frames 2–62 are all delta P-frames).

### Audio Stream
* **Codec:** AAC-LC (FourCC: `mp4a` / `0x6134706D`)
* **Channels:** 1 (Mono)
* **Sample Rate:** 16,000 Hz
* **Duration:** 5.760 seconds (90 AAC frames)
* **Bitrate:** ~17.8 kbps
* **Volume Profile:**
  * `0.00s – 1.80s`: 4 punch/slap audio impulses (gameplay sound effects).
  * `1.85s – 5.76s`: Absolute digital silence (zero amplitude).
  * `5.76s – 6.20s`: Missing stream (audio terminates 440 ms before video).

---

## 4. Root Cause Analysis

### A. The 1-Keyframe Seek Failure
In video compression, P-frames only store differences from previous frames. Because `stss` defines exactly **one** keyframe at index 1, there are no intermediate sync points throughout the entire 6.2 seconds. Seeking to any timestamp forces decoders to either drop frames or decode P-frames without a base reference, causing instant macroblock smear or decoder failure.

### B. Obsolete Codec (`mp4v`) & Modern Hardware Acceleration
Modern GPU decoders (NVIDIA NVDEC, AMD VCN, Intel QuickSync/VA-API) have deprecated hardware decoding for MPEG-4 Part 2. When VLC launches with default hardware acceleration, the VA-API/NVDEC pipeline fails to initialize or drops frame 0 during window creation. Missing frame 0 means missing the *only* keyframe, resulting in a persistent black screen until software fallback (`libavcodec`) is engaged on subsequent runs.

### C. Chromium / Discord Deadlock
Discord's HTML5 media engine relies on synchronized A/V end-of-stream (EOS) signaling. Because audio terminates at `5.760s` while video continues until `6.200s`, and because `moov` is placed at the end of the file without FastStart headers, Chromium's `FFmpegDemuxer` / `GpuVideoDecoder` encounters an unexpected stream state, freezing the player DOM element until destroyed by channel switching.

---

## 5. Byte-Level Forensic Verification

* **Total file size:** `235,864` bytes.
* **`mdat` payload:** `233,384` bytes.
* **Video sample sum (62 packets):** `220,541` bytes.
* **Audio sample sum (90 packets):** `12,843` bytes.
* **Total AV sample coverage:** `220,541 + 12,843 = 233,384` bytes (**100.0% coverage**).
* **Gaps inside `mdat`:** 0 bytes.
* **Trailing overlay bytes:** 0 bytes.
* **Executables / Shellcode signatures:** 0.

---

## 6. Transcoding Fix

To convert this file into a fully compliant, universally playable H.264 MP4:

```bash
ffmpeg -i "TalkingSanta(5)(2).mp4" \
  -c:v libx264 -pix_fmt yuv420p -g 10 -movflags +faststart \
  -c:a aac -ar 44100 -b:a 128k \
  "TalkingSanta_fixed.mp4"
```

---

<details>
<summary><b>Русская версия (Техническое описание)</b></summary>

### Симптомы и причины сбоев

1. **Чёрный экран в VLC при первом запуске:**
   Видеокарта пытается декодировать устаревший кодек MPEG-4 Part 2 аппаратно, для чего в современных GPU нет аппаратных блоков. При инициализации окна плеер пропускает самый первый кадр. Поскольку в видео **всего 1 ключевой кадр на весь ролик**, восстановить последующие кадры невозможно, и плеер отображает чёрный экран до переключения на программный декодер.
2. **Зависание плеера в Discord:**
   Движок Chromium в Discord ожидает корректного завершения обоих потоков. В данном файле звук заканчивается на 5.76 сек, а видео продолжается до 6.20 сек. Медиа-конвейер заходит в состояние взаимной блокировки (deadlock), из-за чего плеер перестаёт реагировать на нажатия до перезахода в канал.
3. **Размазывание пикселей при перемотке:**
   Таблица ключевых кадров `stss` содержит только `[1]`. Любая попытка перемотки приводит к тому, что разностные P-кадры декодируются без базы, создавая артефакты и зависание картинки.
4. **Побайтовая чистота:**
   Все 233 384 байта медиаданных на 100% распределены между 62 видеопакетами и 90 аудиопакетами. Скрытых хвостов, прослоек или шеллкода в файле нет.
</details>
