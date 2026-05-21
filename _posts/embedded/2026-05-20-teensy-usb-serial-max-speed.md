---
layout: post
title: "Teensy 4.1 USB Serial: What is the maximum throughput over USB serial? - Part 2"
date: 2026-05-20
categories: embedded
series: teensy-serial
---

This is part 2 of my series on the Teensy 4.1 USB serial throughput. Read [part 1](https://stpr-dev.github.io/embedded/2026/05/19/teensy-usb-serial-max-speed/) for a bit more background on what I want to achieve. In short, I am trying to benchmark the throughput and reliability of the Teensy 4.1 USB serial interface.

After my recent success, I decided to run a few more frame sizes to see if throughput still goes up or we've peaked. I reran the optimised Python code (the one which precomputes the RNG values). The results are as follows:

| Frame Size (bytes) | Total Time No-ACK (s) | Total Time ACK (s) | Throughput No-ACK (MB/s) | Throughput ACK (MB/s) |
|--------------------|-----------------------|--------------------|--------------------------|-----------------------|
| 128                | 2.9                   | 13.5               | 5.6                      | 1.2                   |
| 256                | 2.9                   | 13.6               | 11.2                     | 2.4                   |
| 512                | 3.5                   | 13.9               | 18.6                     | 4.8                   |
| 1024               | 4.8                   | 14.4               | 27.4                     | 9.2                   |
| 2048*              | 8.3                   | 16.0               | 32.3                     | 16.6                  |
| 4096*              | 16.6                  | 30.4               | 32.3                     | 17.6                  |
| 8192*              | 32.0                  | 64.0               | 32.2                     | 22.2                  |

A few points to note:
- I added (*) to indicate that these runs are flaky. They seem to be susceptible to frame loss, especially more so with the no-ACK mode. It got worse around 4096 and I ran it and higher settings multiple times to get a single successful run.
- We've peaked for no-ACK mode at around 32.3 MB/s. For ACK mode, we're still improving but this isn't entirely surprising. We are essentially getting to a point where we are sending ACK every nth smaller frame, so we are sort of amortising the cost. At the limit, it is essentially no-ACK in disguise.


Now I recognise that my desktop is a bit on the higher-end in terms of specs. I wanted to ensure that this will also apply to slower systems. I found a Thinkpad P16 Gen 1 system and decided to run the tests there.


| Frame Size (bytes) | Total Time No-ACK (s) | Total Time ACK (s) | Throughput No-ACK (MB/s) | Throughput ACK (MB/s) |
|--------------------|-----------------------|--------------------|--------------------------|-----------------------|
| 128                | 6.6                   | 16.0               | 2.5                      | 1.0                   |
| 256                | 6.7                   | 16.8               | 4.9                      | 2.0                   |
| 512                | 6.7                   | 17.2               | 9.9                      | 3.9                   |
| 1024*              | 6.9                   | 20.5               | 19.3                     | 6.6                   |
| 2048*              | 14.1                  | 25.4               | 19.0                     | 10.5                  |

So with the Thinkpad, the peak throughput at a given frame size is significantly lower than on my desktop. Nevertheless, it is still quite respectable. The interesting thing I noted here is that upto 1024 bytes, the total time stays quite constant. This seems to indicate that the overhead for syscall is much higher here than on the desktop case. I did ensure that it was in ultra performance mode. Another thing to note is that the failures are much  frequent in no-ACK mode for sizes >= 1024 bytes. I had to rerun 2048 bytes at least 10 times to get 1 successful run, which was quite a lot. The higher sizes - forget about it, I tried my best but I just couldn't get the no-ACK mode to work. Perhaps the ACK mode would work, but I think I got my answer with this experiment.

So the conclusion so far is that the frame size and throughput depend on the system being used. On my desktop, I can comfortably set 1024 and be fine most of the time. On the Thinkpad, 1024 seems a bit questionable. In either case, some sort of ACK mechanism could significantly improve reliability at the cost of throughput. 

So now that I know generally what performance to expect, there are still 2 questions in my mind:
1. How much performance is left on the table from Python being slow?
2. Does Windows itself set an upper limit on USB serial throughput or reliability?

While 1. is more important for the project, 2. is also personally interesting to me as it gives me an excuse to restart my Linux adventures. But that will have to wait for a bit. In the last post, I mentioned using Go instead of Python. In the past couple of weeks, I had been working on the Go code so now I can share some results!

# Go Rewrite
The Go code is located [in this repository]( https://github.com/stpr-dev/teensy41-serial-companion-go). It is more or less identical to the Python code from my earlier blog. 

Since we are here, I wanted to note my development experience with Go as someone who mostly used Python so far. Now full disclaimer, I used Claude a good amount to help me learn the ropes quickly. Nevertheless, I learnt quite a lot of things:
- Go dependency management is *such a great experience* compared to Python. Things seem to just work. It's easy to point to a GH repo and ask Go to take care of things for you. 
- It was surprisingly easy to pick up Go. There is not much in terms of "fanciness" to the language. I know it's a deliberate design choice by the Go team, and it has people both for and against it. However, personally for me, I found it to be a huge positive - at no point did I feel intimidated by either the build system or the syntax. I could instead focus on the implementation details.
  - Now of course, I do see what people are saying about the whole `if (err != nil) {}` soup. It's a bit verbose, for sure, but it's not something that's a dealbreaker for me. I can live with it. 
- stdlib features seem to rival Python's. I see `flags` which is the `argparse` equivalent, which immediately made it useful for building this tool. Of course, it also has basic stuff like TCP/UDP functionality, but I also got a taste of the HTTP handlers, which were really nice. I didn't use them for this project but have ideas for them. JSON serialization was really interesting and nice.
- Type system is really nice to have – less footguns in general.  
- Single static `.exe` that you can ship to someone and ask them to double-click to run? FINALLY! It is such a relief to not have to ask someone for which version of Python they have installed, and the source (official Python vs conda vs WinPython). Or tell them how to create venv or use `uv` and please not mix dependencies across projects. Or.... you get the idea. 
  - I know `pyinstaller` exists but it's not always great. Static binaries are just sooooo good.
  - I also got a taste for the build flags, which are really useful for controlling compilation.
- Of course, I cannot gloss over Go's most notable feature – concurrency and supporting channels. Coming from Python land, just the thought of **fast** concurrency makes me think, "Oh good, lots of careful design required, probably might involve shared memory and mutexes." I can only describe Go's concurrency model as - a revelation. Just `go myfunc()`? And *directional* channels that are really fast and I don't need to be clever about doing mutexes unless I need to? OH. MY. GOD. Why doesn't every language have this???!
  -  Python does have [Managers](https://docs.python.org/3/library/multiprocessing.html#managers) but my experience with them is that they aren't particularly useful for real-time systems. The abstraction overhead is too high, and I found that just using numpy with SharedMemory and mutexes is much faster.

Ok, enough with my rants, let's take a look at the results when running the Go binary:

| Frame Size (bytes) | Total Time No-ACK (s) | Total Time ACK (s) | Throughput No-ACK (MB/s) | Throughput ACK (MB/s) |
|--------------------|-----------------------|--------------------|--------------------------|-----------------------|
| 128                | 2.8                   | 13.5               | 5.8                      | 1.2                   |
| 256                | 3.3                   | 13.6               | 10.0                     | 2.4                   |
| 512                | 3.7                   | 13.9               | 18.0                     | 4.8                   |
| 1024               | 5.5                   | 14.4               | 24.5                     | 9.3                   |
| 2048               | 8.3                   | 15.5               | 32.3                     | 17.2                  |
| 4096               | 16.6                  | 30.2               | 32.3                     | 17.7                  |

This more or less matches the optimised Python results. Something to note here is that I didn't need to precompute the RNG values like I had to do with Python to achieve these results. I also didn't note as many failed passes with the Go binary as I did with Python, though I feel like I need to do a lot more testing to quantify if this was a fluke. However, the processing headroom is definitely noticeable, and I would be far more comfortable using Go rather than Python for this case.

So this tells me that faster language is beneficial – you have headroom to do stuff like error-checking. But the throughput didn't significantly improve, so we are reaching some limits here. Is it Windows or is it the hardware? Given Paul's (Paul Stoffregen, Teensy's creator) numbers from the last blog post, I feel like we are hitting USB serial limits, though I have to question if the reliability issue is more a Windows thing rather than a USB serial thing. Claude did [find this web page](https://kb.segger.com/CDC), which states:
>Windows 10 comes with a re-designed driver for CDC-ACM. At the time of writing (June 2019) Windows 10 has an issue with large IN CDC transfers. Sometimes packets seems to disappear inside the Windows 10 USB stack. 

Hmm... interesting. This sounds very much like what I am observing. Now I am even more curious to do testing on Linux to validate this theory.

That's the next experiment. Time to grab my live USB!
