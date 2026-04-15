---
title: Integrating High-End Codecs into FFmpeg: JPEG 2000 and R3D
date: 2026-04-15
tags: 
- ffmpeg

---

As part of recent work delivered for a client, we implemented integrations that bring high-performance decoding for JPEG 2000 and R3D media into FFmpeg-based workflows.

This solution is now running in production, enabling efficient handling of professional media formats at scale.

<!--more-->

### Goal

A recurring problem in media systems is the gap between what open-source tooling supports out of the box and what production workflows actually require.

Two formats that regularly surface in high-end pipelines are:

* JPEG 2000 via the Kakadu SDK
* RED R3D via the official RED SDK

Both SDKs provide high-quality, optimised decode paths that are difficult to match with pure open-source implementations. The goal here was to make those capabilities available inside an FFmpeg-based pipeline without compromising on performance or maintainability.

There are a few hard constraints:

* Both SDKs are proprietary
* Both are implemented in C++
* FFmpeg is fundamentally a C codebase

On top of that any integration needs to respect both the SDK licensing terms and FFmpeg’s licensing model.

### Approach

The solution we implemented is based on three layers:

1. FFmpeg (GPL)
2. A thin integration layer (patches)
3. Standalone C++ wrapper libraries around each SDK

#### 1. C++ Wrapper Libraries

We implemented standalone shared libraries that:

* Wrap each proprietary SDK (Kakadu and R3D)
* Expose a minimal, stable **C API**
* Contain all C++ and SDK-specific logic internally

The important design constraint here is that these libraries are **not FFmpeg-aware**. They don’t include FFmpeg headers, and they don’t depend on FFmpeg types or runtime behaviour. From their perspective, they are just decode libraries with a C interface.

#### 2. FFmpeg Integration Patches

On the FFmpeg side, we added patches that:

* Dynamically load the wrapper libraries
* Translate FFmpeg data structures into the wrapper API
* Feed decoded frames back into the FFmpeg pipeline

You can explore the patches here:

* Kakadu integration
  [https://github.com/dalet-oss/FFmpeg/commit/01a23d2b884f0d9a8efb43ffb3bc06083a2a94cb](https://github.com/dalet-oss/FFmpeg/commit/01a23d2b884f0d9a8efb43ffb3bc06083a2a94cb)

* R3D integration
  [https://github.com/dalet-oss/FFmpeg/commit/6b2aadffd13dce41623916f3dfbd69dfb18e24ae](https://github.com/dalet-oss/FFmpeg/commit/6b2aadffd13dce41623916f3dfbd69dfb18e24ae)

Conceptually, this layer is just an adapter: it knows about FFmpeg on one side and a generic C API on the other.

### Licensing & Architecture Considerations

The architecture is intentionally designed so that licensing boundaries align with technical boundaries. A critical requirement was that the wrapper libraries remain **independent artifacts**. In practice, this means:

* They compile only against the vendor SDKs
* They do not include or link against FFmpeg
* They can be built, tested, and shipped separately

To enforce this, we maintain a small set of **CLI tools** that link directly against the wrapper libraries. These act as a basic test harness and are used to validate decoding behaviour independently of FFmpeg.

This is useful not just for testing, but also as a sanity check on the architecture: if the wrapper can’t stand on its own, the boundary is probably wrong.
