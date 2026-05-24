---
layout: post
title: "Teensy 4.1 USB Serial: What is the maximum throughput over USB serial? - Part 4"
date: 2026-05-23
categories: embedded
series: teensy-serial
---

This is part 4 of my series on the Teensy 4.1 USB serial throughput. Read:
- [part 1](https://stpr-dev.github.io/embedded/2026/05/19/teensy-usb-serial-max-speed/) for a bit more background on what I want to achieve. 
- [part 2](https://stpr-dev.github.io/embedded/2026/05/20/teensy-usb-serial-max-speed-part-2/) for benchmark numbers to set some context.
- [part 3](https://stpr-dev.github.io/embedded/2026/05/21/teensy-usb-serial-max-speed-part-3/) for more numbers.

As a followup to part 3, I decided to look into latency numbers as this might provide some insights into the timing jitter. I took the results I logged across the four systems I used in the last experiment and made some plotly plots to see CDF of latency. I also wanted to see if integrating these plots into a GH blog page works seamlessly. Turns out, it is easier than I thought!

Without further ado, here are the results:

<iframe 
  src="/assets/plots/2026-05-23/index.html"
  width="100%"
  height="1000px"
  frameborder="0"
  scrolling="no">
</iframe>

On Windows, zero-latency values show up in large numbers – likely an artefact of the Go reader pulling multiple buffered frames back-to-back, where the system clock doesn't have the resolution to distinguish individual frame arrivals. These aren't meaningful, so I trimmed them (you will see how many were trimmed by looking for the message "X% of samples at or below timing resolution"). Only above-zero values are reflected in the Windows stats. Failed runs are excluded throughout; on Linux there were none.

The main question I had was whether tail latencies were bad enough to matter. They're not — a few millisecond-scale spikes show up at p99.99, but p99 and the median look quite reasonable. The Linux numbers in particular are consistent enough that I'm comfortable calling this interface reliable for my use case.

That wraps up this miniseries. The Teensy USB serial path holds up on Linux: throughput is where it needs to be based on tests from Part 3, and the latency distribution doesn't have anything that would destabilise a real-time pipeline. 
