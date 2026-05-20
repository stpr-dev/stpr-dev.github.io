---
layout: post
title: "Teensy 4.1 USB Serial: What is the maximum throughput over USB serial?"
date: 2026-05-19
categories: embedded
series: teensy-serial
---

As part of a research project, I'm in charge of developing software/firmware for a real-time data acquisition system using [Teensy 4.1](https://www.pjrc.com/store/teensy41.html). This system is interesting in that it has to sample data at several hundred kilohertz, much higher than we typically handle in most of our systems. This comes with its own set of challenges, and I'll detail a few of those across a few posts on this blog. 

For this post, I want to focus specifically on the maximum throughput that one can achieve **reliably** using an out-of-the-box Teensy 4.1 and serial port communication. This is important for me to know to ensure that we'll get the data off of the system as soon as we can without any data loss, and to establish guidelines and test harnesses for the project. Seems like a simple answer, right? After all, Teensy 4.1 [advertises 480 Mbits/s connection](https://www.pjrc.com/store/teensy41.html#specs), and [Paul and other well-known contributors](https://forum.pjrc.com/index.php?threads/recommended-baud-rate-on-teensy-4-1.69839/post-302810) occasionally have explicitly mentioned the same fact:
> Teensy transmits at USB Speed. The baud rate is ignored. And USB Serial on t4.x is very fast (T3.x connects a USB full speed 12MBS, where the T4.x connects at USB high-speed 480mbs)

So the baud rate is ignored (good to know that you can put any number in there, and it doesn't matter), and Teensy 4.1 always connects at 480 Mbits/s. So 480 Mbits /8 = ~60 MB/s, this is what we should be able to achieve (or at least, somewhere in the ballpark) for sustained transfers from Teensy to PC right?

Well, the answer is surprisingly not so clear. In fact, there’s very little information on what's **practically** achievable. To be clear, I fully expected going into this that I won't achieve the maximum speed of 60 MB/s since this is how usually marketing works. The numbers cited are typically *raw* throughput values (which is [technically correct](https://www.youtube.com/watch?v=aIzMuPMicGc)) and don't take into account protocol/OS overheads, so what we *actually* see will be lower than what the marketing numbers are. And of course, it also depends on system load, USB controller quirks, etc. Still, there has to be some value that I should be able to target, give or take. What's that value? 

The only two threads I found that are relevant here are:
1. [This one that explicitly shows some benchmark numbers](https://forum.pjrc.com/index.php?threads/teensy-4-1-coding-guidance-for-usb-serial-communication.76589/) 
>  The Teensy 4.1 supports USB 2.0 High Speed (480 Mbps = 60 MB/s), but I never reach anzywhere close to that.
> 
> My best observed speeds:
PC → Teensy: 7.38 MB/s (using PySerial)
Teensy → PC: 20.97 MB/s (without send_now())

2. [And this comment from Paul](https://forum.pjrc.com/index.php?threads/can-teensy-4-1-bit-stream-at-480mbps-with-usb-2.74782/post-341468)
> The maximum theoretical USB 480 Mbit / second speed with protocol overhead is 53,248,000 bytes/second. See page 55 (83rd page in the PDF) of the USB 2.0 spec for details. But that doesn't include data-dependent bitstuffing overhead or trade-offs all USB host controllers make for bandwidth planning to ensure SOF packets transmit precisely on schedule. So in practice you’ll see even the best USB hardware achieve only some fraction of this theoretical maximum. We’ve often seen about 50% with this benchmark which includes binary to ascii conversion overhead, though results vary quite substantially depending on which software on the PC side receives the data.


OK, so these two do have some numbers to go off of: ~20 MB/s from the first one and ~25 MB/s (from the 50% number Paul mentioned if we take his throughput with overhead included) from the second one (both are surprisingly lower than I expected, which I found interesting, but they still satisfy the project requirements). The first one is that one I'll start with since the stack that I inherited uses `PySerial`. So let's see if I can replicate these numbers.

# Testing
To test this out, I sketched out a minimal test harness that mirrors the project's expected workflow. The core part of the data acquisition system waits for an external trigger, and when the trigger condition is satisfied, the system samples data for T seconds at the given sampling rate ($\ge$100 kHz). Per trigger, we can expect anywhere between $1\times10^{6}$ to $10\times10^{6}$ samples per trigger (the exact numbers depend on the settings, but this is a good ballpark for me to target). So the workload is bursty on a *session* level (since it's mostly idle between triggers), but sustained on a *trigger* level when we're actively acquiring data. So the USB link needs to handle both: long quiet periods and then sudden sustained demand at full rate.

I iterated over the test harness design for a few days and came up with a simple but effective scheme. The full harness is in [this repository](https://github.com/stpr-dev/teensy-usb-test-harness). It is relatively simple — the Teensy is the host and waits for a handshake from the client (in this case, a program running on my PC) of nine bytes, with the following format:
- Bytes 0–3: LE 32-bit integer frame length (M)
- Bytes 4–7: LE 32-bit integer number of frames to send (N)
- Byte 8: Whether to wait for ACK from host per frame sent (0 for no, 1 for yes)

This handshake itself serves as our trigger. This doesn't include the notion of sampling rate since I was hoping to actually measure the upper bound of the max sampling rate that can be achieved. Wait, how would that work? Well, since we're measuring the USB transfer rate, it also indirectly tells us the maximum sampling rate that can be used reliably. If I can send N frames of length M each in a total of T seconds, then the max (amortised) sampling rate is: $\frac{M \times N}{T}$ Hz. This of course assumes `uint8_t` samples, adjust accordingly for different data widths.

Now on the *data* side, I needed a datastream that'll allow me to test that the data I’m receiving is correct: on a byte level per frame, as well as ordering and validity across frames. Now I could have done a repeated pattern of bytes, but that’d be boring. Instead, I decided to do something fun and explore pseudorandom data. I found [this website](https://www.pcg-random.org/posts/some-prng-implementations.html) which has a lot of information on PRNGs in general, but I decided to go with what the website called ["JSF: Bob Jenkins’s Small/Fast Chaotic PRNG"](https://burtleburtle.net/bob/rand/smallprng.html). It seemed simple enough to implement in any language as it mostly appears to contain bitshift operations, which should also be fast on both the PC and Teensy. I also took the opportunity to try out Claude Code for the first time, and asked it to implement `JSF32` in C++ and Python. I tested both of them quickly, and included the C++ version with the firmware. Provided I use the same seed on the client side, it should generate the same outputs, and I can use this to validate samples I am receiving from Teensy on-the-fly. 

So after receiving the handshake, the Teensy now sends a sequence of frames, each containing pseudorandom data generated by the JSF32 PRNG. Each frame (expected to be at least 4 bytes and the size is expected to be a multiple of 4) contains consecutive 32-bit integers (LE) generated by the JSF32 PRNG.

Now that the firmware side is done, I implemented a simple Python client that can send the handshake and receive the data. The client is responsible for a) validating the received data against the pseudorandom sequence generated by the JSF32 PRNG, ensuring data integrity, and b) timing the process to calculate throughput. The client is in [this repository](https://github.com/stpr-dev/teensy-usb-client-python). 

# Results
I did the testing on my desktop with an AMD 7900X PC with 64GB RAM and the USB3 port. The PC should be plenty fast. I initially set a frame length of 2048 bytes and asked for 2^20 frames to be transmitted.

The results were ... *drumroll*... erm, underwhelming? Weird? The client consistently fails validation:

```text
Port COM3 opened.
Starting test. Settings:
Bytes Per Frame: 2048 bytes,
Num Samples: 1048576 samples, Current time: 2026-05-15T14:25:53-06:00
Sent handshake data:
2048 frame size, 1048576 samples, and using ack: false. Waiting for data...
2026/05/15 14:26:11 [Sample 285308, Value 0] Write failed: received 2153709566 value, expected 3948360534
```

Hmm, I wonder if there's an issue with the PRNG implementation? Luckily, the Teensy had an microSD card slot, so I loaded up a microSD, loaded up a simpler firmware that simply wrote around 100k samples to a file. I then copied this to my PC and tested this with my Python implementation. They appeared to agree, so this points to something else going wrong. 

I reran the tests again multiple times, and on a closer observation, I noted that the failures were consistent in the sense that it failed validation during the run, but it didn't always fail at the same *spot*. In other words, `Frame 0 Sample 0` wasn't always the failure point - it was sometimes `Frame 10 or Sample 1024` (interestingly, always a multiple of 512, this will become relevant in other posts). This agaon told me that the PRNG likely wasn't the problem - it was probably `PySerial` or Windows. 

Now I suspect that frames are being dropped. Ok so lets turn on ACK per frame. I fully expect it to kill throughput, [as Paul mentioned](https://forum.pjrc.com/index.php?threads/transfer-to-pc.73502/post-331440), but I just wanted to validate that there was no other issue to blame. Sure enough, the test passed:

```text
# Replace with results
```

As suspected, enabling ACK killed throughput. I was able to achieve ~1.5MB/s with ACK enabled, which is significantly slower than the target I set out to achieve. This tells me that there is indeed an issue with `PySerial` or Windows. Maybe Python is too slow here? I’ll investigate this in a future post.
