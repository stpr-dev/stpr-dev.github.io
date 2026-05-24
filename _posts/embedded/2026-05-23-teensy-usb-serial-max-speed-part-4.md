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

As a followup to part 3, I also decided to look into latency numbers as that matters for real-time transfer. I just took the results and made some plotly plots to see CDF of latency. Without further ado, here are the results:

<iframe 
  src="/assets/plots/2026-05-23/index.html"
  width="100%"
  height="1000px"
  frameborder="0"
  scrolling="no">
</iframe>

Something to note: on Windows, I trimmed zero values which are reported as being "X% of samples at or below timing resolution". There were lots of zero values which made the analysis annoying so I decided to get rid of them.
