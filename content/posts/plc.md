---
title: MAPNACTF 2024 - PLC Challenge
description: Writeup for the PLC network challenge involving packet analysis
date: 2024-01-20
tldr: Analyzing a pcap file to reconstruct a fragmented flag from network traffic.
draft: false
tags: ["ctf", "writeup", "Network", "forensics"]
---

# PLC

## Challenge Overview

The challenge involved analyzing a pcap (packet capture) file, typically used in network analysis, to find and reconstruct a flag.

## Approach

We used Wireshark, a popular network protocol analyzer, to inspect the contents of the provided pcap file. The challenge here was to identify and piece together fragments of a flag scattered across network packets.

### Finding Flag Fragments

We utilized the `strings` command in combination with `grep` to extract potential flag fragments from the pcap file:

```bash
strings plc.pcap | grep '^[0-9]:' | sort
```

This command sequence searches for strings in the pcap file, filters those that look like parts of our flag (starting with a number and a colon), and sorts them.

### Flag Fragments Identified

The output provided us with the following fragments:

```
1:MAPNA{y
2:0U_sHOu
3:Ld_4lW4
4:yS__CaR
5:3__PaAD
6:d1n9!!}
```

## Flag Reconstruction

By carefully assembling these fragments in order, we reconstructed the complete flag:

```
MAPNA{y0U_sHOuLd_4lW4yS__CaR3__PaADd1n9!!}
```

## Conclusion

This challenge highlighted the importance of thorough packet analysis in network forensics. By examining the pcap file, we were able to identify and piece together fragmented data to successfully extract the flag.
