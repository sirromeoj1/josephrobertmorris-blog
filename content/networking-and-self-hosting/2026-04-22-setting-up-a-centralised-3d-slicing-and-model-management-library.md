---
title: Setting Up a Centralised 3D Slicing system in a VM and a 3D Model Library using Manyfold
description: "How I set up a Virtual Machine for centrlised 3D Slicing and Manyfold for managing a database of 3D Models"
date: 2026-04-22T20:16:15.698Z
preview: ""
draft: false
tags: [3D Printing, Manyfold, Slicing, Virtual Machine, Truenas, SliceVM]
categories: []
showComments: true
---

**Initial Problems I Wanted to Solve**

I have been 3D printing since 2023 when I received an Ankermake M5 FFF 3D printer I had backed on Kickstarter months earlier. Over the next few years I built up quite the library of 3D models scattered across various folders — or just dumped on my desktop — that I had haphazardly tried to organise at various points using various systems. Everything was always a faff to find, often to the point where I was searching for and redownloading things I knew I had saved somewhere. I needed a better system. I needed Manyfold.
Manyfold is a self-hosted library management tool for 3D files. It has some great features for organising your model library: collections, tags, fields for information like author and licence, and it can import models directly from a URL for some repositories like Thingiverse (sadly not Printables or Makerworld, the two best places online to find models). This is all backed up by decent search and 3D previews directly in the browser. It's a pretty great, if slightly buggy, piece of software.
Another problem I had always had was creating and syncing filament and print profiles across the various computers and operating systems I had PrusaSlicer and OrcaSlicer installed on, and having them survive system reinstalls and the death of hard drives. Since I wanted to play around with virtual machines anyway, and a VM is actually a great solution to this problem, I decided to set one up on my TrueNAS box.

**Installing Dockge and Manyfold**

I deployed Manyfold as a Docker container using Dockge. Installing Dockge went off without a hitch — HexOS has it listed as an unsupported app, but following a few online guides and ChatGPT got it set up very quickly. After that, ChatGPT produced a Docker Compose script for Manyfold which worked first try. This part of the process was genuinely easy.
I did run into frequent hanging when uploading a lot of models from my library, which I never fully resolved. It only crops up when uploading dozens of models at once, so I don't consider it a big deal for my use case. Examining the logs revealed a 500 error occurring when the hang happened, but I never got any further than that.

**The Arduous Journey of Setting Up a VM**

**Part 1 — Why You Practically Need a GPU**

Without a GPU, a TrueNAS VM uses software rendering on the CPU through a program called llvmpipe. This works fine for everyday tasks, but for 3D slicing software like PrusaSlicer or OrcaSlicer it is utterly inadequate, leading to slideshow-level performance when viewing G-code or rotating the camera in the editor. Although a more powerful CPU than my Ryzen 5 5700g may improve this even a low-end or integrated GPU is almost always going to be substantially better than any CPU at graphics workloads.

Given the need for a GPU, there are a few things worth keeping in mind. I would strongly recommend reserving a GPU for TrueNAS itself in case something goes wrong and you need to directly interact with the machine. Because of this, I suggest having two GPUs in your system — one for TrueNAS and one reserved for your VM. In my case, the intergrated GPU on my Ryzen 5700g is for truenas and a Raeoon WX3200 is reserved for the VM. You can lock any hardware device to a specific VM in TrueNAS settings.
Another consideration is PCIe bandwidth, slot size, and allocation. Consumer-grade motherboards often have limited PCIe slots, these are frequently shared with other devices like SATA ports or NVMe slots that may be disabled when a PCIe card is installed, are often routed through the chipset rather than the CPU which can cause performance issues with some devices, and often have physically closed ends so you can't fit a card with a longer edge connector than the slot itself. Why manufacturers do this last one is genuinely beyond me. Make sure you have somewhere sensible to put your GPU alongside any other PCIe devices you may have, like an HBA card for additional drives.
One more thing: you will almost certainly need an HDMI or DisplayPort dummy plug. Consumer GPUs generally won't properly initialise their rendering pipeline without an active display output — they think nothing is connected and don't fully wake up. A dummy plug is a small adapter that tricks the GPU into thinking a monitor is plugged in. Without one, you may find the GPU is visible to the system but refuses to accelerate anything.

**Part 2 — The Linux Failure**

Whilst I'm sure it is possible to get a slicing VM with GPU acceleration working perfectly well on Linux to say I ran into issues is an understatement. There were too many to cover in detail here, and sadly I did a very poor job of documenting them. A few examples of the types of problems I encountered: Wayland preventing Remote Desktop Protocol from working, programs using the wrong device for hardware acceleration, the GPU failing to initialise properly on VM startup, and Wayland preventing shared clipboard access between the VM and my desktop machine. Some of these have fixes, some may be resolved in future Linux or driver releases, and some can be diagnosed quickly by asking an AI assistant or searching a forum. I just kept running into one isses after another at every stage. 

**Part 2.5 — i440FX vs Q35**

One potentially major problem worth calling out separately, because I only discovered it quite late in my journey: TrueNAS sometimes uses a virtual chipset called i440FX by default. This is a recreation of an Intel chipset from 1996. For most VM use cases this is perfectly fine, but for PCIe passthrough of devices like a GPU it can cause serious problems. i440FX is fundamentally a PCI machine type — it predates PCIe entirely and can only emulate it. The mismatch can mean the guest operating system can see the GPU but cannot properly initialise it as an accelerated device. For GPU passthrough, the newer Q35 machine type is recommended. As far as I can tell, changing this needs to be done in the TrueNAS shell rather than the UI.

Honestly I don’t know how much on a issue this was for me specially, I ran into a lot of problems that could have been caused by this, but I didn’t discover it much later in the process so I cannot say. I think it is defiantly worth doing however.

**Part 3 — The Eventual Solution: Windows (HUUUURGGEHHH) 11 (HUUUURGGEHHHHHHH)**

As a recent escapee from Microsoft's advertisement generator and AI slop shoveller disguised as an operating system to Bazzite Linux, I was very reluctant to return. However, Windows is better supported for this use case, and both PrusaSlicer and OrcaSlicer run slightly better on Windows in my experience, so here we are.
Setup wasn't particularly difficult. One thing to be aware of is that Windows has no built-in drivers for the paravirtualised disk and network devices that the virtual machine software (QEMU) presents to the VM — without installing the VirtIO drivers, the Windows installer simply cannot see the installation drive at all. Once those were sorted, installation proceeded normally.
For remote access I chose Sunshine and Moonlight. Sunshine runs on the VM and captures the GPU's output directly, streaming it to a Moonlight client on another machine. Because it captures from the real GPU rather than a virtual display, you get proper hardware-accelerated rendering at full resolution — none of the software rendering limitations that affect RDP. Setup was mostly straightforward, with one notable gotcha: accessing the Sunshine web interface from another machine on your network will initially throw a CSRF protection error. CSRF (Cross-Site Request Forgery) is a web security mechanism, and Sunshine blocks requests coming from a different origin than localhost by default. The fix is to add origin_web_ui_allowed = lan to the Sunshine config file, which allows access from any device on your local network. This only affects the configuration interface — the actual Moonlight streaming connection is unaffected.

**Part 4 — Using It and What's Next**

Using this setup has been great. Not having access to a filament profile because I made it on my laptop, or failing to find a model I downloaded five months ago — those problems are in the past. Everything is in one place and accessible from any machine.

A few things I'd like to improve in the future:

I would like the print and filament profile folders for OrcaSlicer and PrusaSlicer to be automatically backed up using something like symlinks to my NAS, so I don't lose anything if the VM stops working for whatever reason.
I would like to back up my Manyfold library to an external service like Proton Drive, or to another NAS using Buddy Backup on HexOS, to make sure I never lose that library.
Performance is still a little laggy. I'm sure more tuning could be done, and I haven't enabled Hardware Accelerated GPU Scheduling (HAGS) yet, which may improve smoothness.
I would like to get audio working. There are apparently several ways of doing this but I haven't investigated yet. 
