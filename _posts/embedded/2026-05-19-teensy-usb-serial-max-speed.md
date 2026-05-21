---
layout: post
title: "Teensy 4.1 USB Serial: What is the maximum throughput over USB serial?"
date: 2026-05-19
categories: embedded
series: teensy-serial
---

As part of a research project, I'm in charge of developing software/firmware for a real-time data acquisition system using [Teensy 4.1](https://www.pjrc.com/store/teensy41.html). This is the first time I got to work with a Teensy, and I have to say, I am pretty impressed with the capabilities of the tiny, yet fierce, device. I am also really glad that Teensyduino supports C++17 so I can work on a slightly more familiar (and sometimes better IMO) C++ standards rather than plain C code (though sometimes looking at the maze of weird C++ quirks makes me appreciate C a lot).

Anyway, the data acquisition system is interesting in that it has to sample data at several hundred kilohertz, much higher than we typically handle in most of our systems. This comes with its own set of challenges, and I'll detail a few of those across a few posts on this blog. The core part of the data acquisition system waits for an external trigger (a button press, for instance), and when the trigger condition is satisfied, the system samples data for T seconds at the given sampling rate ($\ge$100 kHz). Per trigger, we can expect anywhere between $1\times10^{6}$ to $10\times10^{6}$ samples (the exact numbers depend on the settings and are being finalised, but this is a good ballpark for me to target). So the workload is bursty on a *session* level (since it's mostly idle between triggers), but sustained on a *trigger* level when we're actively acquiring data. So the USB link needs to handle both: long quiet periods and then sudden sustained demand at full rate. Of course, this isn't *entirely* accurate, as there are things like DMA that come into the picture, but we can ignore that for now as it doesn't meaningfully change the math at the moment.

For this post, I want to focus specifically on the maximum throughput that one can 
achieve **reliably** using an out-of-the-box Teensy 4.1 and serial port communication. For now, I am discounting any custom USB stuff and focusing on getting data out of the default COM port. This is important for me to know so that I can ensure that the data is drained from the Teensy quickly, preventing any data loss due to buffer overruns. I can use this to establish guidelines like max sample rate, max duration, analogue-to-digital converter resolution, etc. I can also provide a basic test harness for the project so that people can test reliability of the transfer. 

Seems like a simple answer, right? After all, Teensy 4.1 [advertises 480 Mbits/s connection](https://www.pjrc.com/store/teensy41.html#specs), and [Paul and other well-known contributors](https://forum.pjrc.com/index.php?threads/recommended-baud-rate-on-teensy-4-1.69839/post-302810) (Paul Stoffregen, Teensy's creator) occasionally have explicitly mentioned the same fact:
> Teensy transmits at USB Speed. The baud rate is ignored. And USB Serial on t4.x is very fast (T3.x connects a USB full speed 12MBS, where the T4.x connects at USB high-speed 480mbs)

So the baud rate is ignored (good to know that you can put any number in there, and it doesn't matter), and Teensy 4.1 always connects at 480 Mbits/s. So 480 Mbits /8 = ~60 MB/s, this is what we should be able to achieve (or at least, somewhere in the ballpark) for sustained transfers from Teensy to PC right?

Well, the answer is surprisingly not so clear. In fact, there’s very little information on what's **practically** achievable. To be clear, I fully expected going into this that I won't achieve the maximum speed of 60 MB/s since this is how usually marketing works. The numbers cited are typically *raw* throughput values (which is [technically correct](https://www.youtube.com/watch?v=aIzMuPMicGc)) and don't take into account protocol/OS overheads, so what we *actually* see will be lower than what the marketing numbers are. And of course, it also depends on system load, USB controller quirks, etc. Still, there has to be some value that I should be able to target, give or take. What's that value? 

Two threads I found particularly relevant here are:
1. [This one that explicitly shows some benchmark numbers](https://forum.pjrc.com/index.php?threads/teensy-4-1-coding-guidance-for-usb-serial-communication.76589/) 
>  The Teensy 4.1 supports USB 2.0 High Speed (480 Mbps = 60 MB/s), but I never reach anzywhere close to that.
> 
> My best observed speeds:
PC → Teensy: 7.38 MB/s (using PySerial)
Teensy → PC: 20.97 MB/s (without send_now())

2. [And this comment from Paul](https://forum.pjrc.com/index.php?threads/can-teensy-4-1-bit-stream-at-480mbps-with-usb-2.74782/post-341468)
> The maximum theoretical USB 480 Mbit / second speed with protocol overhead is 53,248,000 bytes/second. See page 55 (83rd page in the PDF) of the USB 2.0 spec for details. But that doesn't include data-dependent bitstuffing overhead or trade-offs all USB host controllers make for bandwidth planning to ensure SOF packets transmit precisely on schedule. So in practice you’ll see even the best USB hardware achieve only some fraction of this theoretical maximum. We’ve often seen about 50% with this benchmark which includes binary to ascii conversion overhead, though results vary quite substantially depending on which software on the PC side receives the data.

(Also, shoutout to this [interesting thread](https://forum.pjrc.com/index.php?threads/high-speed-usb-data-custom-isochronous-usb-descriptor.64481/), that seems very similar to what I am doing. Some interesting ideas for custom USB descriptors and settings, but that is something I will look at perhaps in the future).

OK, so these two do have some numbers to go off of: ~20 MB/s from the first one and ~25 MB/s (from the 50% number Paul mentioned if we take his throughput with overhead included) from the second one (both are surprisingly lower than I expected, which I found interesting, but they still satisfy the project requirements). The first one is that one I'll start with since the stack that I inherited uses `PySerial`. So let's see if I can replicate these numbers.

# Testing
To test this out, I sketched out a minimal test harness that mirrors the project's expected workflow. 

I iterated over the test harness design for a few days and came up with a simple but effective scheme. The full harness is in [this repository](https://github.com/stpr-dev/teensy41-serial-test-firmware). It is relatively simple — the Teensy waits for a handshake from the PC (more specifically, a Python script). The handshake is expected to be nine bytes, with the following format:
- Bytes 0–3: LE 32-bit integer frame length (M)
- Bytes 4–7: LE 32-bit integer number of frames to send (N)
- Byte 8: Whether to wait for ACK from host per frame sent (0 for no, non-zero for yes)

This handshake itself serves as our trigger. This doesn't include the notion of sampling rate since I was hoping to actually measure the upper bound of the max sampling rate that can be achieved. Wait, how would that work? Well, since we're measuring the USB transfer rate, it also indirectly tells us the maximum sampling rate that can be used reliably. If I can send N frames of length M each in a total of T seconds, then the max (amortised) sampling rate is: $\frac{M \times N}{T}$ Hz. This of course assumes `uint8_t` samples, adjust accordingly for different data widths.

Now on the *data* side, I needed a datastream that'll allow me to test that the data I’m receiving is correct: on a byte level per frame, as well as ordering and validity across frames. Now I could have done a repeated pattern of bytes, but that’d be boring. Instead, I decided to do something fun and explore pseudorandom data. I found [this website](https://www.pcg-random.org/posts/some-prng-implementations.html) which has a lot of information on PRNGs in general, but I decided to go with what the website called ["JSF: Bob Jenkins’s Small/Fast Chaotic PRNG"](https://burtleburtle.net/bob/rand/smallprng.html). It seemed simple enough to implement in any language as it mostly appears to contain few bitshift and logical operations, which should also be fast on both the PC and Teensy. I also took the opportunity to try out a couple of things I've been wanting to do:
- Setup [Platform IO for Teensy](https://docs.platformio.org/en/latest/platforms/teensy.html)  for [CLion](https://www.jetbrains.com/clion/). I am a massive fan of JetBrains IDEs (PyCharm/CLion/Rider, et al.) and have been using it for a while now.
- Use Claude Code for the first time to do some basic automated coding for simple tasks.
- Fully integrate [uv](https://docs.astral.sh/uv/) on Python side for package management and dependency resolution. Big fan of `uv`, it really solved a major  annoyance I have with `pip` and `requirements.txt` files. Most of the stuff I did with it were on existing projects, so I figured it would be a good time to try it out on a new project.

I asked CC to implement `JSF32` in C++ and Python. I tested both of them quickly to ensure that they provided the same outputs. I included the C++ version with the Teensy firmware. As long as I use the same seed on the client side, it should generate the same outputs, and I can use this to validate samples I am receiving from Teensy on-the-fly. 

After receiving the handshake, the Teensy now sends a sequence of frames, each containing pseudorandom data generated by the JSF32 PRNG. Each frame (expected to be at least 4 bytes and the size is expected to be a multiple of 4) contains consecutive 32-bit integers (LE) generated by the JSF32 PRNG. Fairly straightforward.

Now that the firmware side is done, I implemented a simple Python client that can send the handshake and receive the data. The client is responsible for a) validating the received data against the pseudorandom sequence generated by the JSF32 PRNG, ensuring data integrity, and b) timing the process to calculate throughput. The Python script is in [this repository](https://github.com/stpr-dev/teensy41-serial-companion-python). 

# First Pass
I used the basic example for this. No real optimization was done as I felt that the code was relatively straightforward (spoiler alert: probably not).

## Results
I did the testing on my desktop with an AMD 7900X PC with 64GB RAM and the USB3 port. The PC should be plenty fast. I initially set a frame length of 2048 bytes (value picked from [this post](https://forum.pjrc.com/index.php?threads/increase-the-usb-buffer-size-in-teensy-4-1.73656/post-332404) and asked for $2^{17} = 131,072$ frames to be transmitted.

The results were ... *drumroll*... erm, underwhelming? Weird? My script indicates that it fails validation:

```text
uv run .\pyserial_example.py --port "COM3" --frame-size 2048 --num-frames 131072
Port COM3 opened.
Sent handshake data:
2048 frame size, 131072 frames, and using ack: false ...
...
Traceback (most recent call last):
  ...
ValueError: [Frame (8), Value (0)] Write failed: Received 1663108126 value instead of expected 261443033.
```

One thing I noticed that was it didn't fail right away. That is a good indication that it wasn't likely that the random numbers are wrong, but more likely that data is being corrupted during transmission. 

I ran it again, and it failed again, though at a different spot:

```text
ValueError: [Frame (119034), Value (0)] Write failed: Received 2472662199 value instead of expected 195615389
```

Now I suspect one of two things: 
a) Python is too slow so there is buffer overrun, or
b) The cable or transmission is faulty

I suspect a) is likely true. One easy fix is to turn on ACK per frame. I fully expect it to kill throughput, [as Paul mentioned](https://forum.pjrc.com/index.php?threads/transfer-to-pc.73502/post-331440), but I just wanted to validate that there was no other issue to blame.

Aha! So this passed:

```text
...
Min delays: 448300.00 ns
Max delays: 1294100.00 ns
Average delays: 522467.98 ns
```
It is indeed buffer overrun. I ran this multiple times, and it passed every time.

## Reruns
The ACK method indeed felt slow, it took a long time. But it bugged me that the no-ACK version didn't pass. So I backed out a little and decided that I would try different frame lengths. Maybe it could reveal something?

I wrote a PowerShell script to automate it. Surprisingly, a few of them ran with no issues. The results were:

| Frame Size (bytes) | Total Time No-ACK (s) | Total Time ACK (s) | Throughput No-ACK (MB/s) | Throughput ACK (MB/s) |
|--------------------|-----------------------|--------------------|--------------------------|-----------------------|
| 128                | 4.4                   | 13.9               | 3.7                      | 1.2                   |
| 256                | 7.2                   | 15.5               | 4.6                      | 2.1                   |
| 512                | 13.0                  | 28.4               | 5.1                      | 2.3                   |
| 1024               | 24.8                  | 40.9               | 5.4                      | 3.2                   |
| 2048               | —                     | 68.4               | —                        | 3.9                   |

Oh. **Very interesting**. The no-ACK version does indeed pass, and it seems to consistently pass at *lower* frame lengths. This was very surprising to me: I thought that lower frame lengths would be worse due to syscall overhead. Seems like I was wrong here. However, two things stood out to me:
1. The ACK version's throughput is half as fast as the no-ACK version.
2. Even the no-ACK version's throughput is substantially lower than what I was targetting.

Now technically, per project requirements, 5MB/s would work, so the 512 no-ACK version is sufficient for our needs. However, I was still not entirely happy with the throughput of the no-ACK version, as it was significantly lower than the target. I decided to investigate further to see if there were any optimizations that could be made to improve the performance.

# Second Pass
When rerunning the tests, I noticed something interesting. I used the LED indicator on the Teensy to tell me when it started/stopped sending the data. While this was happening, I was observing the console output and noted that the Teensy finished significantly before the console output stopped when using the no-ACK version. This got worse with increasing frame lengths, suggesting that the Teensy was able to send the data faster than Python could keep up with.

So I started digging through the Python code. The only bottleneck I could think of was the RNG. Though it was extremely simple, I decided to benchmark it. Here's what I found:

```text
Average time per sample: 664.8639 ns
```

Doesn't seem like a lot but if we were to multiply that by 512, we are looking at 338 microseconds per frame, which doesn't sound like much but it is in the same order of magnitude as the processing time per frame. So this could potentially be a bottleneck. 

I copied the script and made one modification: I pre-generated the values I was expecting ahead of time. I then reran the tests. Now while running it, I did notice that it took a significant amount of time to generate the values so there is indication that the RNG is indeed the bottleneck. 

Regardless, here are the results of the tests with the pre-generated values:

| Frame Size (bytes) | Total Time No-ACK (s) | Total Time ACK (s) | Throughput No-ACK (MB/s) | Throughput ACK (MB/s) |
|--------------------|-----------------------|--------------------|--------------------------|-----------------------|
| 128                | 2.9                   | 13.5               | 5.6                      | 1.2                   |
| 256                | 2.9                   | 13.6               | 11.2                     | 2.4                   |
| 512                | 3.5                   | 13.9               | 18.6                     | 4.8                   |
| 1024               | 4.8                   | 14.4               | 27.4                     | 9.2                   |
| 2048               | 8.3                   | 16.0               | 32.3                     | 16.6                  |


And there it is! Few interesting points to note:
- With frame sizes >=1024 and no-ACK, we were able to achieve >25MB/s. 2048 seems the maximum stop but the tests were flaky - sometimes they failed. It wasn't 100% reproducible but of the 10 runs I ran, around 30% failed. 
- Sizes <=512 seem to take about the same time, indicating potential syscall overhead.
- The ACK version still has quite poor throughput, anywhere from 3-5x worse. 

# Conclusion
Technically speaking, it is indeed possible to achieve around 30MB/s with no-ACK and frame sizes >=1024. At least on my PC, on a laptop or slower systems, it will be lower. I confirmed this myself by running this on a slower laptop and I had much lower throughputs. 

However, there is a glaring flaw here: while I could  pre-compute the values for this case, in real production cases, this isn't feasible. Given the flakiness of the protocol, I think that some minimal framing is necessary to ensure reliable communication. At minimum, something like sequence number and CRC of the frame should be included to ensure data integrity and reliability. This poses a problem with Python: if JSF is too slow, CRC or other error detection methods will almost definitely be much slower. Additionally, we didn't even do any of the other stuff that comes with the acquisition: UI, API calling, IO, etc. All of these are things you would see in the full application. 

The results don't fill me with confidence that the Python approach will work in a real-world setting. Not in the very least without substantial architecting around using multiprocessing to decouple the reading loop from the processing loop. And boy,isn't multiprocessing IPC fun to work with in Python (being sarcastic BTW).

This is one of those cases where I think that a faster language would be the better solution. I've been meaning to learn and use Go for something real-world. Guess this is my calling? Time to learn Go!
