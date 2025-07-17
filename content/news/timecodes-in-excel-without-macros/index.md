---
title: Timecodes in Excel without Macros
date: 2019-06-17
tags: 
- excel
- timecode 

---

Googling for "excel timecode" didn't produce any results which met the requirements:

- support 29.97 drop-frame SMPTE timecode
- calculate a duration given an in-point and out-point
- provide conversion to and from timecode, frame count and microseconds
- not based on Excel macros so that the solution can be easily redistributed

Because of this, I am pleased to present a solution which does meet the above requirements: 

[https://github.com/vectronic/timecode-spreadsheet](https://github.com/vectronic/timecode-spreadsheet)

<!--more-->
