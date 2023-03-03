---
title: "My LG OLED TV is crashing"
date: 2023-03-03T13:31:27Z
draft: false
tags: 
  - LG TV 
  - OLED
  - Jumbo Frames
  - Crash
  - Wireshark
categories: 
  - Software Glitches
---

I love my LG OLED TV, I believe the picture quality is unbeatable and I could call myself an LG TV advocate. However, this is not to say they're flawless. I like WebOS, but's it becoming clunky and slow - there's too many advertisements all over the place and the UI is not as responsive as it used to be - this is said based on my experiences of owning two C1 models of 55 and 65 inch size. It sometimes frustrates me that starting Disney Plus can take almost a minute if not longer when initiated via the remote shortcut when the TV is off.

What really did stumble me is the rare occurrence when the TV would randomly freeze and restart. I've observed it a couple of times and it was always when I was watching a movie or a TV show. I've tried to reproduce it, but it was not easy - I went for the process of elimination as eventually, the issue was occurring when nothing was being watched.

## Process of elimination

As the issue was reproducible on both of my TVs and it happened at the same time, I went ahead with the process of elimination and started off with the network. To my surprise, the issue disappeared - strange, as I wouldn't say there is anything special on my network that could cause it.

Next, knowing the network has something to do with the crashes, I started observing the pattern and found that the issue has tendency to happen when my work laptop is on but with the screen off and idling. Knowing that the specific setup I had involved having jumbo frames enabled on the interface, I wondered if it could have anything to do with it.

I disabled jumbo frames on the interface for a period of a few days and the issue disappeared - this system was the only one device in the network that was up and running most of the time due to... reasons that deserve another post. Yes Lenovo - I'm talking about you 🤦

## Jumbo frames

Jumbo frames, whilst enabled are not of much usefulness to me right now - yet I wanted to figure out what particularly upsetting the TV so I resorted to my good friend Wireshark.

Eventually, I isolated two types of packets that were below the standard MTU:

- KDE connect packets broadcasted to UDP port `1716` and length of 2034 bytes
- Network Discovery packets originating from port `5357` length MTU of 2401 bytes

![Screenshot of Wireshark capture showing packets of size larger than 1500 MTU](wireshark.png)

## Why LG, why?

With this bizarre issue identified, perhaps I should bring it to LGs attention?
