---
layout: post
title: "Teensy 4.1 USB Serial: What is the maximum throughput over USB serial? - Part 3"
date: 2026-05-21
categories: embedded
series: teensy-serial
---

This is part 3 of my series on the Teensy 4.1 USB serial throughput. Read:
- [part 1](https://stpr-dev.github.io/embedded/2026/05/19/teensy-usb-serial-max-speed/) for a bit more background on what I want to achieve. 
- [part 2](https://stpr-dev.github.io/embedded/2026/05/20/teensy-usb-serial-max-speed/) for benchmark numbers to set some context.

In short, I am trying to benchmark the throughput and reliability of the Teensy 4.1 USB serial interface.

I ended part 2 wondering if there is some jank with Windows and USB serial throughput/reliability. I've established what I think is an upper limit on Teensy -> PC USB throughput using my Go code. I was still not entirely convinced that the reliability was good, so I decided to do three things today:
- Increase the number of frames sent to better assess reliability.
- Run the test harness on Linux to see if there is any differences in throughput/reliability.
- Run the test harnesses on four different computers using both Windows and Linux on each computer to see if the results I was observing last time were fluke or there is actual merit to them.

My tests in the last blogs asked Teensy to send 131,072 frames, which for lower frame sizes only lasted single-digit seconds. I decided to 8x that to 1,048,576 frames, which I expected to take longer. This should be stressing the system a bit more to hopefully expose any dropped packets. 

For Linux (I know it's a kernel and not the entire OS, but I am using the colloquial term here), I download the lastest [Arch Linux](https://archlinux.org/download/) (btw) ISO (2026-05-01 image) and loaded it onto a USB drive with [Rufus](https://rufus.ie/) (v4.14). I also copied over the static Go binary from [this release](https://github.com/stpr-dev/teensy41-serial-companion-go/releases/tag/v0.1.0-alpha) to the USB drive. I didn't bother with the Python version because:
- setting up Python for live Linux is not something I am interested in subjecting myself to when the awesome Go binary is available, and 
- pre-allocation would not be feasible for these sizes (it will require several gigabytes of RAM for pre-allocation). 
For each computer I tested, I used the default Windows 11 install that was on it, and then once it was done, rebooted to the live USB and ran the test from the terminal.


The four computers I used for the test consist of:
- Two high-performance desktops: one with AMD 7900X and the other with AMD 9800X3D. One thing to note is that both use different motherboards/chipsets - I wanted to mention this since I know that they are both AM5-based CPUs so they can hypothetically use the same motherboard/chipset. Two different chipsets would at least give some variability to isolate any weird chipset-specific IO quirks. 
- Two laptops: one Lenovo Thinkpad and one Lenovo Yoga Book, both using Intel CPUs. 

This gives me a good variety of relatively high-end desktops to an arguably low-powered Yoga Book, and CPUs/chipsets from both major vendors. I am going to focus on the no-ACK version (why I chose to only focus on this will become clear very soon). 

Now let’s take a look at the results. First, let's look at the 7900X desktop:


| Frame Size (bytes) | Throughput Windows (MB/s) | Throughput Linux (MB/s) |
|--------------------|---------------------------|-------------------------|
| 128                | 5.1                       | 19.3                    |
| 256                | 10.2                      | 28.6                    |
| 512                | 18.3*                     | 28.8                    |
| 1024               | 24.6*                     | 27.9                    |
| 2048               | 32.7*                     | 36.9                    |
| 4096               | 32.7*                     | 36.9                    |

Few things to note here:
- I used the same machine and USB ports for this, and the same Go code cross-compiled to both Windows and Linux.
  - Now, of course, just because I cross-compiled it doesn't mean that the low-level behaviour is guaranteed to be the same. The syscalls and other OS primitives around data transfer will be different across OSes. It just means that the overall logic is consistent. 
- The Linux results are remarkably faster than windows across all frame sizes. 
- Unlike Windows, Linux doesn't appear to heavily penalise smaller frame sizes. On Linux, the min throughput for the smallest was 19.3 MB/s compared to a max of 36.9 MB/s. That is only 50% slower. On Windows, the min is 5.1 MB/s, which is about 6x slower than the max of 32.7 MB/s. 
- The peak throughput was also higher for Linux than for Windows, though not substantially (32.7 MB/s vs 36.9 MB/s).

Now notice the (*) on Windows' numbers? That is because I saw failures. That's right; unlike last time, I saw a failure on frames as small as 512:

```text
Sent handshake data:
512 frame size, 1048576 frames, and using ack: false. Current time is 2026-05-21T12:33:09-06:00. Waiting for data...
2026/05/21 12:33:09 Running simple loop-based acquisition.
2026/05/21 12:33:19 [Sample 369082] Failed to read full frame: read returned 0 bytes after 0/512
```

That's disappointing. It appears to have timed out for the read. You know what is even more surprising? The Linux tests **never** saw any failures, even for the smallest frame sizes. Not once. I ran the tests multiple times and still didn't see any failures. 

OK, this was maybe a fluke. Let's look at a different machine. Here is the 9800X3D one:

| Frame Size (bytes) | Throughput Windows (MB/s) | Throughput Linux (MB/s) |
|--------------------|---------------------------|-------------------------|
| 128                | 2.5                       | 18.3                    |
| 256                | 5.1                       | 27.5                    |
| 512                | 10.2*                     | 26.2                    |
| 1024               | 19.1*                     | 30.7                    |
| 2048               | 32.4*                     | 36.0                    |
| 4096               | 32.4*                     | 36.0                    |


These results are worse on Windows compared to the last time. The smaller frame sizes are even slower here (2.5MB/s vs 5.1 MB/s on the last PC). On Linux, the pattern is more or less the same as the first machine, and I saw no errors across multiple runs. 

I even encountered my first weird error on Windows:

```markdown
2026/05/21 13:37:10 Retrying read after error: read returned 0 bytes after 13/1024
```

What? **13 bytes**? That's a really odd number (both linguistically and mathematically). I am used to seeing multiples of 512, or at least powers of 2, which would make sense. 13 is really odd. But let's keep going.

Next up, the Thinkpad:


| Frame Size (bytes) | Throughput Windows (MB/s) | Throughput Linux (MB/s) |
|--------------------|---------------------------|-------------------------|
| 128                | 2.5                       | 11.3                    |
| 256                | 5.1                       | 8.6                     |
| 512                | 10.2*                     | 14.7                    |
| 1024               | 18.9*                     | 22.2                    |
| 2048               | -                         | 31.2                    |
| 4096               | -                         | 31.2                    |

For the Thinkpad, I was unable to get the test to pass at frame sizes larger than 1024 bytes on Windows. Same as before, I saw errors from 512 frame size onwards on Windows, but on reruns, I was able to get past it. On Linux, the tests chugged along with no errors. The Thinkpad was actually approaching desktop-level performance on USB with Linux, something I didn't see on Windows. I even made sure to turn on "ultra performance mode" on the laptop to disable any energy-saving behaviour on Windows. 


Finally, the Yoga Book. This one was more of a curiosity for me as the device is not as high-performance as the other devices, but I wanted to see how it would perform to provide recommendations for the project.

| Frame Size (bytes) | Throughput Windows (MB/s) | Throughput Linux (MB/s) |
|--------------------|---------------------------|-------------------------|
| 128                | -                         | 13.5                    |
| 256                | -                         | 8.4                     |
| 512                | -                         | 14.3                    |
| 1024               | -                         | 21.0                    |
| 2048               | -                         | 30.8                    |
| 4096               | -                         | 30.8                    |

Oooh boy. So **none** of the Windows tests passed. I tried running my PowerShell script multiple times, but it always failed right at the beginning at 128 bytes of frame length. I get two errors on Windows:

```markdown
2026/05/21 15:25:33 Retrying read after error: read returned 0 bytes after 13/128
```

Once again, our friend 13-byte error. Why 13?!!! 

Or the more classic error, which indicates full frames being dropped:

```markdown
2026/05/21 15:26:34 [Sample 448065] Failed to read full frame: read returned 0 bytes after 0/128
```

Linux once again passed with flying colours, no errors reported and really good throughput.

# Conclusion
So what have I learnt so far?

First, let me answer the question that I used to title this miniseries: What is the maximum USB serial throughput for a Teensy 4.1 with a USB? Or more precisely:
- If you are continuously dumping data from Teensy with the intent of reading that data from a PC connected to the Teensy, and 
- Without sending an ACK back

what is the throughput you expect to see? Based on measurements across four machines, it is somewhere between **30MB/s - 36MB/s**. Some nuance to this figure, as we've seen in these posts, mainly around _reliability_.

The number above is assuming **ideal conditions** where you don't lose any data. I've learnt that Windows is absolutely terrible in this regard. I was able to get significantly better performance on every machine with Linux. Heck, laptops running Linux could achieve almost desktop-like throughputs. You could absolutely critique my method of using an Arch Live USB commandline-only environment vs a full desktop Windows, which I grant is absolutely fair. For these tests, the machines I was working with were work machines, so I couldn't install Linux with a full Desktop Environment like KDE. At some point, perhaps I will try the tests out on a machine that I own with and without a DE running to see how much impact that has. I suspect that I will be seeing little of a difference in the results. So **if you want to achieve max performance and no dropped data, Linux appears to be mandatory**.

For Windows, I suspect that we will end up needing to design a protocol at the application layer to detect data corruption and implement retransmission logic. It is an open question what this looks like or how best to implement it. There are also alternative avenues to explore here: custom USB drivers, Ethernet adapters, using *actual* serial instead of USB serial, etc. I will keep them in mind as I settle on a solution that works for the project I am working on.

The 13-byte partial reads on Windows are interesting – I hypothesise that the Windows USB CDC drivers are terminating the reads early. I will put this on my list of things to investigate further.

Overall, I am happy with this investigation. The next step for me is to evaluate other parts of this acquisition system, including DMA and SD writes. These will be part of my upcoming posts!


# Footnotes
- Side rant: I am always annoyed by secure boot and having to disable it to boot from the live USB. I wish it were plug-and-play, but this isn't Arch Linux's fault – it is just the way secure boot works.
