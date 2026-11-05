---
title: Time-addressable Media Store (TAMS) support for FFmpeg
date: 2026-11-05
tags:
- ffmpeg
- tams

---

We have been working on adding support for the [Time-addressable Media Store](https://bbc.github.io/tams/) (TAMS) to FFmpeg: a demuxer that reads Flows and Sources from a TAMS store, and a muxer that writes to one, both live and non-live.

The work is available on the `tams` branch of our FFmpeg fork, as a patch series which we are preparing to submit upstream.

<!--more-->

### What is TAMS?

TAMS is an open API specification, started by BBC R&D and now developed with a growing group of broadcasters and vendors, for storing media in the cloud as small time-addressable chunks rather than as monolithic files.

The core concepts are:

* **Sources** - an abstract piece of content, e.g. "the camera 1 feed" or "episode 3"
* **Flows** - a concrete representation of a Source, e.g. "1080p H.264 at 8Mbps" or "stereo AAC"; Flows can be video, audio, data, image or *multi* (a collection of other Flows)
* **Segments** - Media Objects (a small MPEG-TS, MP4, raw elementary stream etc.) registered against a Flow for a given timerange
* **Timeranges** - everything is addressed on a nanosecond TAI timeline, e.g. `[1792591960:585251000_1792591966:585251000)`

Because media is stored as immutable, independently addressable segments on a shared timeline, it becomes possible to start working with content while it is still being recorded, to make edits by reference instead of by copy, and to share objects between Flows.

The API spec is here: [https://github.com/bbc/tams](https://github.com/bbc/tams), and AWS provide an open-source implementation: [https://github.com/aws-samples/time-addressable-media-store](https://github.com/aws-samples/time-addressable-media-store).

### Why FFmpeg?

A store is only useful if tools can get media in and out of it. Having TAMS supported natively in FFmpeg means anything which uses FFmpeg (ffmpeg, ffprobe, ffplay, and the many applications built on libavformat) can read from and write to a TAMS store simply by using a URL:

```shell script
ffmpeg -headers "Authorization: Bearer $TOKEN" \
  -i "https://tams.example.com/flows?source_id=<sourceId>" ...

ffmpeg -i INPUT -c:v libx264 -c:a aac \
  -flow_map "flow=streams=0,1,multi_flow=0" \
  -headers "Authorization: Bearer $TOKEN" \
  -f tams https://tams.example.com/flows
```

### The Branch

The code is on the `tams` branch here:

[https://github.com/hootworks/FFmpeg/tree/tams](https://github.com/hootworks/FFmpeg/tree/tams)

It is structured as three patches, as it would be submitted to the FFmpeg mailing list:

1. `avformat/tams`: shared code for the TAMS data model
2. `avformat`: the TAMS demuxer
3. `avformat`: the TAMS muxer

The main files are:

* `libavformat/tams.h` / `tams.c` - the TAMS data model (`TAMSFlow`, `TAMSFlowSegment`, `TAMSTimeRange`, `TAMSContainerMapping` etc.), parsing and serialising Flows, Segments and storage allocations to and from JSON, timestamp and timerange handling, codec MIME mapping, and HTTP requests with retry
* `libavformat/tamsdec.c` - the demuxer
* `libavformat/tamsenc.c` - the muxer
* `libavformat/tests/tams*.c` - unit tests, run as part of FATE
* `doc/demuxers.texi` / `doc/muxers.texi` - documentation

### Demuxer

The demuxer accepts either a single Flow (`/flows/<id>`) or all the Flows of a Source (`/flows?source_id=<id>`), optionally limited to a timerange. It creates one `AVStream` per video/audio/data Flow, expanding multi Flows into their members.

Each set of segments is read through a nested demuxer: a `TAMSSegmentContext` owns a child `AVFormatContext` which opens one segment at a time, with packet timestamps mapped from the segment's container timeline onto the Flow timeline. Where several Flows share one container (a multi Flow with a `container_mapping`) the segment context is shared and packets are routed back to the right stream by track index, MPEG-TS PID, ISOBMFF track id, or MXF track id.

```mermaid
classDiagram
    direction TB

    class TAMSFlow {
        id : char[37]
        source_id : char[37]
        format : TAMSFlowFormat
        codec : char[128]
        container : char[128]
        timerange : TAMSTimeRange
        segment_duration : AVRational
        status : char[32]
        nb_flow_collection_items : int
        essence parameters...
    }

    class TAMSFlowCollectionItem {
        id : char[37]
        role : char[64]
        has_container_mapping : int
    }

    class TAMSContainerMapping {
        track_index : int
        mp2ts_pid : int
        isobmff_track_id : int
        mxf_track_id : int
    }

    class TAMSFlowSegment {
        object_id : char[512]
        timerange : TAMSTimeRange
        ts_offset : int64_t
        get_url : char[2048]
    }

    class AVFormatContext
    class AVStream
    class AVFormatContext_sub_ctx

    class TAMSDemuxContext {
        nb_flows : int
        nb_seg_ctxs : int
        live_threshold_s : int64_t
        live_timeout_s : int64_t
        min_segment_buffer_s : int64_t
    }

    class TAMSStreamContext {
        flow_index : int
        parent_flow_index : int
        seg_ctx_index : int
        sub_stream_index : int
        next_pts_ns : int64_t
    }

    class TAMSSegmentContext {
        flow_index : int
        nb_flow_segments : int
        cur_flow_segment_index : int
        is_live : int
        poll_interval_us : int64_t
    }

    TAMSFlow "1" *-- "0..16" TAMSFlowCollectionItem : flow_collection_items
    TAMSFlowCollectionItem --> "0..1" TAMSContainerMapping : container_mapping
    AVFormatContext --> TAMSDemuxContext : priv_data
    AVFormatContext --> "*" AVStream : streams
    AVStream --> TAMSStreamContext : priv_data
    TAMSDemuxContext --> "*" TAMSFlow : flows
    TAMSDemuxContext --> "*" TAMSSegmentContext : seg_ctxs
    TAMSStreamContext ..> TAMSFlow : flows[flow_index]
    TAMSStreamContext ..> TAMSSegmentContext : seg_ctxs[seg_ctx_index]
    TAMSSegmentContext --> "*" TAMSFlowSegment : flow_segments
    TAMSSegmentContext --> AVFormatContext_sub_ctx : sub_ctx
```

And the sequence of calls each time the caller asks the demuxer for a packet (`tams_read_packet`):

```mermaid
sequenceDiagram
    participant CLI as ffmpeg CLI
    participant Dec as tamsdec.c
    participant Sub as nested demuxer
    participant TAMS as TAMS store
    participant Obj as object storage

    CLI->>Dec: read_packet
    Dec->>Dec: pick the stream (not at EOF) with the lowest current_ts_ns
    opt no segment open for its segment context
        Dec->>Dec: tams_ensure_segments
        opt fewer than min_segment_buffer seconds of segments listed
            Dec->>TAMS: GET /flows/{id}/segments?timerange=... (next page)
            opt live, at the live edge and nothing new
                Dec->>TAMS: GET /flows/{id}?include_timerange=true (status, segments_updated, timerange)
                Note over Dec: wait, doubling the wait up to seg_poll_max
            end
        end
        opt no segments left
            Dec->>Dec: mark the context's streams EOF, pick again
        end
        Dec->>Sub: tams_open_segment: avformat_open_input(get_url)
        Sub->>Obj: GET segment object
        Dec->>Dec: resolve sub-streams, validate, copy extradata
    end
    loop until a packet is returned
        Dec->>Sub: av_read_frame
        alt end of segment object
            Dec->>Sub: close, move to the next segment (open as above)
        else packet
            Dec->>Dec: tams_map_packet_timestamps (to the Flow timeline)
            Dec->>Dec: tams_classify_packet
            alt OK
                Dec-->>CLI: packet
            else DISCARD
                Dec-->>CLI: packet flagged AV_PKT_FLAG_DISCARD
            else EOSEG
                Dec->>Sub: close, move to the next segment
            else EOF
                Dec->>Dec: mark the stream EOF, pick again
            end
        end
    end
```

### Timeranges

Timeranges turned out to need more care than expected, so it is worth describing how the demuxer handles them.

**The forms.** A TAMS timerange is `[start_end)` with `[`/`]` inclusive and `(`/`)` exclusive markers, and either end may be missing. There are also some special forms:

| Form | Meaning | Written by FFmpeg as |
|------|---------|----------------------|
| `[5:0_10:0)` | from 5 s up to (not including) 10 s | `[5:0_10:0)` |
| `[5:0_)` | from 5 s onwards | `[5:0_)` |
| `_` | eternity: all time | `(_)` |
| `[5:0]` | instantaneous: exactly 5 s | `[5:0_5:0]` |
| `()` | never: the empty range | `()` |

`TAMSTimeRange` keeps a separate `never` flag rather than encoding the empty range as some start and end, so "empty" and "unbounded" can't be confused. `ff_tams_timerange_intersect()` takes the later start and the earlier end, with each end inclusive only if both ranges include it, and gives never if the ranges do not overlap.

**A Flow's timerange isn't always in the response.** The spec only returns a Flow's timerange from `GET /flows/{id}` when `include_timerange=true` is given, and `GET /flows` has no way to ask for it at all. The demuxer can't change the first request (FFmpeg opens the URL and probes the response before the demuxer is chosen), so for a Flow without a timerange it makes a second request with `include_timerange=true`. A Flow without a timerange in the response is different from one whose timerange is `()`: the first is unknown, the second means the Flow has no segments yet, e.g. a live Flow which has just been created. A multi Flow whose media is all in its members' containers is also always `()`.

**Limiting a read to a timerange.** A `timerange=` in the opened URL limits what is read. It means different things to the store depending on the URL:

* on `/flows/{id}` it only limits the timerange returned for the Flow (and only when `include_timerange=true` is also given)
* on `/flows?source_id=...` it only filters which Flows are returned, so a Flow which overlaps the range at all is returned with its full timerange. A multi Flow's own `()` never overlaps, so its members need to be asked for by `/flows/{id}`

So the demuxer applies the URL's timerange itself. Each Flow's timerange, as given by the store, is narrowed to its intersection with the URL's. This is done for the Flows in the first response, for Flows fetched to find their timerange, for multi Flow members fetched afterwards, and again whenever a live Flow is re-fetched while polling for new segments:

```mermaid
flowchart TD
    A["Flow from the store"] --> B{"timerange in the response?"}
    B -- "no" --> C["GET /flows/{id}?include_timerange=true"]
    C --> B
    B -- "yes, but it is never" --> K{"live?"}
    K -- "yes" --> D["use the URL timerange: wait for segments in it"]
    K -- "no" --> H
    B -- "yes" --> E["intersect with the URL timerange"]
    E --> F{"overlap?"}
    F -- "yes" --> G["read the intersection"]
    F -- "no" --> J{"live, and the URL timerange is after the Flow's end?"}
    J -- "no" --> H["read nothing"]
    J -- "yes" --> I["use the URL timerange: poll until the Flow reaches it"]
```

The narrowed timerange then does several jobs:

* it gives each stream its duration
* it bounds the segment requests (`/flows/{id}/segments?timerange=...`), so segments outside it are never listed or downloaded. Later pages start after the last segment seen, and keep the end if reading stops there
* it sets where a stream ends. A live Flow's end is only how far it has been written so far, so a live read stops at the end of the URL's timerange, or follows the Flow until it closes if there is none

**Segments aren't cut on the requested range.** Segments start on keyframes, so a requested start usually falls part way through a segment. The packets before the start are still returned, but flagged `AV_PKT_FLAG_DISCARD`: a decoder needs the preceding keyframe and the frames after it to decode the first frames in range, and drops the output of the flagged ones. Decoding `[40.0_40.5)` from a Flow with a single segment covering 39.74 to 40.70 s therefore returns exactly 0.5 s of frames, while `-c copy` keeps the flagged packets so that the copy is still decodable.

The nested demuxer can set the discard flag itself, e.g. an MP4 edit list trims the open-GOP leading pictures before an object's first keyframe. Within a Flow those pictures continue the previous segment, so the demuxer clears that flag and decides which packets to discard only from the segment and Flow timeranges.

### Muxer

The muxer writes each input `AVStream` to a Flow. By default every stream gets its own mono Flow, but the `-flow_map` option can group streams into a multi Flow sharing one container, or pin streams to existing Flows and Sources so that a new recording is appended to an existing timeline.

Each segment is produced by a nested muxer (raw elementary stream, MP4, MOV, MPEG-TS or Matroska), uploaded to a presigned storage URL obtained from the store, and then registered against the Flow with its timerange.

```mermaid
classDiagram
    direction TB

    class TAMSMuxContext {
        flow_map_str : char*
        container_name : char*
        segment_duration_s : int64_t
        start_tai_ns : int64_t
        nb_container_ctxs : int
        nb_multi_flow_ctxs : int
    }

    class TAMSContainerContext {
        oformat : const AVOutputFormat*
        owner_flow_id : char[37]
        reference_stream_index : int
        segment_start_ns : int64_t
        next_boundary_ns : int64_t
        next_put_url : char*
    }

    class TAMSFlowContext {
        flow_id : char[37]
        source_id : char[37]
        flow : TAMSFlow
    }

    class TAMSMultiFlowContext {
        flow_id : char[37]
        label : char[256]
        flow : TAMSFlow
    }

    class AVFormatContext
    class AVStream
    class AVFormatContext_sub_ctx

    AVFormatContext --> TAMSMuxContext : priv_data
    AVFormatContext --> "*" AVStream : streams
    TAMSMuxContext --> "*" TAMSContainerContext : container_ctxs
    TAMSMuxContext --> "*" TAMSMultiFlowContext : multi_flow_ctxs
    TAMSContainerContext --> "*" TAMSFlowContext : flow_ctxs
    TAMSContainerContext --> AVFormatContext_sub_ctx : sub_ctx
    TAMSMultiFlowContext ..> TAMSContainerContext : container_indices[j]
```

And the sequence of events when the muxer starts:

```mermaid
sequenceDiagram
    participant CLI as ffmpeg CLI
    participant Mux as tamsenc.c
    participant Sub as nested muxer
    participant TAMS as TAMS store

    CLI->>Mux: write_header
    Mux->>Mux: validate streams (codec MIME + essence parameters)
    Mux->>Sub: build one nested muxer per container
    Mux->>TAMS: GET pinned Flows / PUT new Flows (status ingesting)
    Mux->>TAMS: GET /service
    Mux->>TAMS: POST /flows/{id}/storage (presigned URL for segment 0)
    CLI->>Mux: write_packet ...
    Mux->>Sub: packets until the first keyframe after segment_duration
    Mux->>TAMS: PUT segment bytes, POST /flows/{id}/segments
    CLI->>Mux: write_trailer
    Mux->>TAMS: last segment, PUT Flows (status closed_complete)
```

### Interesting Points

A few things which came up during the implementation and testing:

**No JSON library.** FFmpeg has no JSON parser dependency, so the JSON handling is a small cursor-based parser covering just the TAMS schemas, built on `AVBPrint`.

**Segments must be self-contained.** The first version of the muxer kept a single nested muxer running and simply swapped its output buffer at each segment boundary. That works for MPEG-TS, but not for MP4 (where the `moov` only appears at the end) or raw formats whose headers only appear once. The muxer now restarts the nested muxer for every segment, in the same way as FFmpeg's `segment` muxer.

**Raw segments need their parameter sets.** The TAMS muxer declares global headers, as MP4 and Matroska need the codec's parameter sets (H.264/HEVC SPS and PPS, the MPEG-2 sequence header) when they write their own header, so encoders put them only in the stream's extradata. A raw elementary-stream segment has no header, so the muxer copies the extradata in front of every keyframe. That is only right when the extradata is already in start-code form: when H.264 or HEVC is stream copied from MP4, it is in MP4's own `avcC`/`hvcC` form, and the raw muxer's own conversion filter already inserts the parameter sets. Testing only with encoded input had hidden this, so stream copy is now part of the tests.

**Segments are cut on keyframes, so timeranges must be too.** Segments can only start at a keyframe, so with x264's default GOP of 250 frames a 2 second `segment_duration` really means 10 second segments. Registering the *planned* boundary rather than the actual cut point meant most of each segment was discarded on playback. This was the cause of a stuttering live stream which kicked off the latest round of testing.

**`ts_offset` depends on the container.** TAMS defines a segment's `ts_offset` as the value added to each container timestamp to put it on the Flow timeline. Raw elementary streams restart from zero in every segment, while MP4, MPEG-TS, Matroska and raw FLAC keep continuous timestamps (with their own shifts, e.g. MPEG-TS adds twice the mux delay). The muxer accounts for each case. When a segment has no `ts_offset` but its media timestamps lie outside the segment's timerange (e.g. objects with timestamps starting near 0 registered against a TAI timerange), the demuxer infers the offset from the first packet, as otherwise every packet would fall outside the segment.

**Following a live Flow.** A reader can't know whether a Flow is still being written just from its segments. TAMS 8.2 adds a Flow `status`, which the muxer sets to `ingesting` while writing and `closed_complete` when done. The demuxer follows it, falling back to timing heuristics (based on `segments_updated` and the observed segment length) for Flows without a status, or whose writer has disappeared.

**Segment boundaries in shared containers.** When video and audio share a container which is cut on a video keyframe, a few audio packets (and, with open GOP HEVC, a few leading pictures) belong to the *previous* segment's time but live in the next object. The demuxer keeps packets that follow on from what has already been delivered, rather than discarding everything outside the segment's timerange.

**Credentials.** Authentication uses standard FFmpeg `-headers`, and those headers are only replayed to the store's own host, never to the presigned storage URLs.

### What is Supported

* TAMS API version 8.2
* Demuxing a Flow or all Flows of a Source, optionally limited to a timerange, with seeking
* Live demuxing, including joining late, and ending when the writer closes the Flow or stops
* Video, audio, data (subtitle) and multi Flows; image Flows for demuxing
* Multi Flows with shared containers (`container_mapping`) or independent members
* Muxing to new Flows/Sources, or appending to existing Flows, Sources and multi Flows
* Video codecs: H.264, HEVC, VP8, VP9, AV1, MPEG-2, raw video
* Audio codecs: AAC, MP3, MP2, AC-3, E-AC-3, FLAC, Opus, Vorbis, PCM (16/24/32-bit integer, 32/64-bit float)
* Subtitles: SubRip, WebVTT
* Containers: raw elementary streams, MP4, MOV, MPEG-TS, Matroska (plus MXF for demuxing)

### What is Not Supported (yet)

* Flows using separate initialisation segments (`init_object`, new in 8.2) - these are rejected with a clear error
* Flow profiles, webhooks, and Flow/Source tag and label editing beyond what is set at creation
* Obtaining or refreshing OAuth tokens: a bearer token must be supplied via `-headers`
* Raw video or PCM segments with no container: the demuxer detects segment formats by probing rather than from the Flow's `container`, and these have nothing to probe (the muxer stores them in Matroska/MP4)
* Muxing to MXF or Ogg (Ogg does not split cleanly into independent segments, so Vorbis uses Matroska)
* MP4 multi Flows containing AAC are placed about 21ms late on the Flow timeline (A/V sync is unaffected)
* Seeking with `-ss` in Flows stored as raw H.264 or HEVC elementary streams is only accurate to the segment: these streams carry no timestamps, so ffmpeg cannot discard the frames between the start of the segment and the seek point, and output starts at the beginning of the segment containing the seek point. Flows in a container (MP4, MPEG-TS, Matroska) seek accurately

### Testing

The patches include unit tests for the JSON parsing, timestamp and timerange handling, the demuxer's segment, timestamp and URL timerange logic, and the muxer's `-flow_map` parsing, along with FATE tests demuxing TAMS samples, and the full FATE suite passes.

The FATE samples are store responses and segment objects saved as local files. A local input is read as a set of complete responses: no query strings are added, there is a single page of segments and Flows are not re-fetched, so the file names contain no `?` and work on Windows. One sample Flow has a timerange which starts and ends part way through its segments, to test discarding before the start and stopping at the end. The behaviour which needs a live store (applying the URL's timerange, building the segment requests and finding where a read ends) is covered by the unit tests.

In addition, everything was tested end-to-end against a hosted instance of the AWS open-source TAMS implementation (API 8.2):

* Muxing a 6 second test source with each supported video and audio codec, then demuxing by Flow and by Source, checking every frame is present and timestamps are continuous, and the same with stream copy from MP4 and MPEG-TS input
* Each container, multi Flows (shared and independent), and pinning to existing Flows, Sources and multi Flows, including appending a second recording to an existing timeline without a gap
* Timerange requests and seeking
* Live: muxing a 720p test source in real time while reading it back concurrently, by Flow and by Source, with different GOP sizes, containers and codecs, joining late, and with the muxer killed part way through
* Reading existing third-party content from the store
* Fuzzing the demuxer with damaged JSON

Not yet tested: MXF and image Flows (no test content), very long running live recordings, and stores other than the AWS implementation.

### Next Steps

The next step is to submit the patches to the FFmpeg development mailing list for review. If you are using TAMS and would like to try this out, or have content we could test against, please get in touch.
