title: 网络性能benchmark
date: 2014-12-13 20:07:03
tags: driver
categories: 编程
---
##RFC 2544
###9.1 Frame sizes to be used on Ethernet
64, 128, 256, 512, 1024, 1280, 1518
These sizes include the maximum and minimum frame sizes permitted by
the Ethernet standard and a selection of sizes between these extremes
with a finer granularity for the smaller frame sizes and higher frame
rates.
<!-- more -->
##RFC 5180
Appendix A. Theoretical Maximum Frame Rates Reference
This appendix provides the formulas to calculate and the values for
the theoretical maximum frame rates for two media types: Ethernet and
SONET.
###A.1. Ethernet
The throughput in frames per second (fps) for various Ethernet
interface types and for a frame size X can be calculated with the
following formula:
Line Rate (bps)
——————————
***(8bits/byte)*(X+20)bytes/frame**
The 20 bytes in the formula is the sum of the preamble (8 bytes) and
the inter-frame gap (12 bytes). The throughput for various Ethernet
interface types and frame sizes:

|Size|10Mb/s|100Mb/s|1000Mb/s|10000Mb/s|
| ----- | :----- | :----- | :----- | :----- |
|64     |14,880  | 148,809 | 1,488,095 |14,880,952|
|128    |8,445   | 84,459 | 844,594| 8,445,945|
|256    |4,528   | 45,289 | 452,898| 4,528,985|
|512    |2,349   | 23,496 | 234,962| 2,349,624|
|1024   |1,197   | 11,973 | 119,731 | 1,197,318|
|1518   |812     | 8,127 | 81,274 |  812,743|

###Ethernet size
Preamble 64 bits
Frame 8 x N bits
Gap 96 bits
###Note:
Ethernet’s maximum frame rates are subject to variances due to
clock slop. The listed rates are theoretical maximums, and actual
tests should account for a +/- 100 ppm tolerance.
