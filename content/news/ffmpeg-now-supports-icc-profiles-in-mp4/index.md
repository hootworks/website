---
title: FFmpeg now supports ICC Profiles in MP4 
date: 2020-03-16
tags: 
- ffmpeg

---

Last year I provided some FFmpeg patches to support [ICC Profiles](http://www.color.org/specification/ICC1v43_2010-12.pdf) 
stored within MP4 (and MOV) files.

The patches were recently merged into master and will therefore be available in the next FFmpeg release.

`\( ﾟヮﾟ)/`

ICC profiles can be stored in MOV/MP4 sample descriptor colour information atoms. 
The relevant extract from the [ISO standard](https://standards.iso.org/ittf/PubliclyAvailableStandards/c068960_ISO_IEC_14496-12_2015.zip) is:

<!--more-->

```
class ColourInformationBox extends Box('colr'){
    unsigned int(32) colour_type;
    if (colour_type == 'nclx')  /* on-screen colours */
    {
        unsigned int(16) colour_primaries;
        unsigned int(16) transfer_characteristics;
        unsigned int(16) matrix_coefficients;
        unsigned int(1)  full_range_flag;
        unsigned int(7)  reserved = 0;
    }
    else if (colour_type == 'rICC')
    {
        ICC_profile;    // restricted ICC profile
    }
    else if (colour_type == 'prof')
    {
        ICC_profile;    // unrestricted ICC profile
    }
}
```

FFmpeg already had support for the `nclx` colour type, but my use case was to support the `prof` colour type.

Additionally, ICC profile support was already available on a per frame basis in 
`AVFrameSideDataType.AV_FRAME_DATA_ICC_PROFILE`. This is used by the PNG, WEBP and MJPEG codec implementations.
This support was not sufficient however as the MOV/MP4 sample descriptor relates to the entire stream and not on a per-frame basis.

#### Changes

The changes required were to:

* [add a new enum value](https://github.com/FFmpeg/FFmpeg/commit/05d27f342be28cf92f3c9470e701834c416cad89) `AV_PKT_DATA_ICC_PROFILE` to `AVPacketSideDataType` 
* add reading support to the MP4 [demuxer](https://github.com/FFmpeg/FFmpeg/commit/472e044587d5fe43bbc84713ca824fd8350924b8)
* add writing support to the MP4 [muxer](https://github.com/FFmpeg/FFmpeg/commit/dc1c3c640d245bc3e7a7a4c82ae1a6d06343abab)

The last item included adding a movflag `prefer_icc` to ensure backwards compatible behaviour of the existing `write_colr` movflag.

#### Usage

Here is a small example file with an ICC Profile embedded: [icc-profile.mp4](http://vectronic.io/icc-profile/icc-profile.mp4)

With the above changes in place and using the sample file, when you execute `ffprobe` you will see the output includes a new side data entry of `ICC Profile`:

```shell script
% ./ffprobe -hide_banner icc-profile.mp4
Input #0, mov,mp4,m4a,3gp,3g2,mj2, from 'icc-profile.mp4':
  Metadata:
    major_brand     : mp42
    minor_version   : 1
    compatible_brands: mp41mp42isom
    creation_time   : 2019-09-19T08:13:20.000000Z
  Duration: 00:00:00.08, start: 0.000000, bitrate: 32519 kb/s
    Stream #0:0(und): Video: hevc (Main) (hvc1 / 0x31637668), yuv420p(tv, progressive), 3440x1440 [SAR 1:1 DAR 43:18], 31699 kb/s, 60 fps, 60 tbr, 600 tbn, 600 tbc (default)
    Metadata:
      creation_time   : 2019-09-19T08:13:20.000000Z
      handler_name    : Core Media Video
    Side data:
      ICC Profile
```

When remuxing an MP4, the following will preserve the ICC profile in the output:

```shell script
./ffmpeg -i icc-profile.mp4 -movflags write_colr+prefer_icc -c:v copy out.mp4
```
