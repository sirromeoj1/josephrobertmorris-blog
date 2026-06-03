---
title: Modernising a CD collection on Linux, Fre:ac, EasyTAG, MusicBrainz/freedb/GnuDB, Jellyfin and DLNA/UPnP
description: "Using Fre:ac to rip cds and broadcasting them over my network via DLNA"
date: 2026-06-03T00:56:09.267Z
preview: ""
draft: false
tags: [Linux, Bazzite, Jellyfin, DLNA, Fre:ac, EasyTAG, CD ripping, FLAC, TrueNAS, home media server]
categories: []
showComments: true
---

Recently my Grandma passed away, leaving us with an extensive collection of CDs, few of which we have room for. So I decided that I would rip them to my computer. However, one thing I have found throughout the years is that, unless you have an easy way to play them on the devices that you regularly use for music listening, they will simply rot on your hard drive.

The devices used in our household to listen to music are mostly phones and a Denon Ceol RCD-N11DAB media centre. I wanted a minimal software setup that could play on both. This would need to be reliable, hassle-free and easy to use. I chose Jellyfin for this task; it has a perfectly cromulent Android app that can stream music from my TrueNAS server (and download for offline use), and has a DLNA/UPnP server that the Ceol and HEOS (the phone app used to control of the Ceos) can pick up.
DLNA (Digital Living Network Alliance) is essentially a multimedia flavour of UPnP (Universal Plug and Play), a set of protocols allowing devices on a network to identify each other and figure out what each device can do. This can be done without IP configuration; if both devices are on the same network (foreshadowing noises) and both support UPnP, then they should be able to spot and identify each other. This is done via messages, sent via multicast, announcing what each device can do and where it is.
The Setup
Given the large number of CDs in the collection, this needed to be as seamless as possible with minimal manual intervention. So I decided on this: I would use Fre:ac to record the CDs. It can record FLAC files for lossless recording and can import CD tags like track name, artist, album, etc. from various databases like GnuDB. It then outputs its files, with the correct folder structure, to Jellyfin's music folder, which slurps them up and broadcasts them using its DLNA server.

Music databases are very inconsistent in how exactly they tag things in my experience, especially for Classical music. For example, some tracks may put the performers of the piece in the artist field, some may put the composer in there, and some people may call the same composer different things, for example Bach, J.S. Bach, Bach J S, Johann Sebastian Bach, etc. Since DLNA uses these tags as a way to organise your music collection, it is vital to have these correct and, more importantly, consistent; otherwise, as your collection gets larger, it will get harder to find what you want. I chose EasyTAG for editing these tags manually to keep them consistent with the rest of my library.

I edited Fre:ac's folder structure to more closely match Jellyfin's default; I used <artist>/<album>/<track>-<title>. This can be done in Options - General Settings, Output Files.

**Installing and setting up Jellyfin, Fre:ac, EasyTAG**

This part was easy; the Bazzite store had both Fre:ac and EasyTAG, so installation was painless. Since my TrueNAS server is running HexOS (an easier-to-use wrapper for TrueNAS), I just used their one-click install for Jellyfin (this would cause problems later). After a brief setup screen, I installed the DLNA server plugin for Jellyfin.
Next came mounting Jellyfin's Music folder on my Bazzite desktop. I went with a CIFS mount, as these can be more performant than alternatives like GVFs and can be made permanent in immutable distros like Bazzite. First I needed to install CIFS and reboot.

**rpm-ostree install cifs-utils**
**systemctl reboot**

Next I created a new folder and mounted Jellyfin's Music share there.

**mkdir -p ~/Music-Server**
**sudo mount -t cifs //Your-Server's-IP-address/Music ~/Music-Server**
**-o username=YOURUSER,uid=(id−u),gid=(id -u),gid=**
**(id−u),gid=(id -g),vers=3.0**

Then to make it permanent:

**sudo bash -c 'cat > /etc/cifs-credentials-joenas <<EOF**
**username=YOUR TRUNAS USER**
**password=YOUR TRUNAS PASSWORD**
**EOF'**
**sudo chmod 600 /etc/cifs-credentials-joenas**

Then edit fstab using:

**sudo nano /etc/fstab**

Add this line at the bottom of the document:

**//Your-Server's-IP/Music  /home/yourusername/Music-Server  cifs  credentials=/etc/cifs-credentials-joenas,uid=1000,gid=1000,vers=3.0,nofail,x-systemd.automount  0  0**
Press CTRL O, Enter to save, then Ctrl X to exit. The nofail and x-systemd.automount are important here; they prevent hanging if TrueNAS is unreachable, and make sure the mount occurs when first accessed.
Then after that, give EasyTAG and Fre:ac permission to access that folder:

**flatpak override --user --filesystem=home/Music-Server org.freac.freac**
**flatpak override --user --filesystem=home/Music-Server org.gnome.EasyTAG**

**Dealing with Networking, Docker/Kubernetes and HEOS issues**

So at this point, everything was set up: I had recorded a couple of CDsand gotten the Jellyfin Music share mounted and persisting across reboots. I just had one issue: the DLNA server was not showing up in the HEOS app. I had a quick look using VLC; no luck there either. After looking at the Jellyfin logs, it turned out it was broadcasting on completely the wrong network. WTF?
Basically, TrueNAS creates an internal network for containers, which Jellyfin happily broadcasts its DLNA messages on. To fix this, Jellyfin needs to be set to host mode networking in the TrueNAS app settings page.
This made it show up on the Denon CEOL itself, and in VLC, but not in the HEOS interface. This turned out to be a device-level issue; it only doesn't show up on my phone (Samsung A35 5G); it shows up on the Pixel 7A. I have no idea why and have been unable to solve this issue. It can see the Jellyfin server so I doubt it's a networking issue; I tried disabling Private DNS in Samsung Connection settings, which didn't fix it. The Pixel is the main one I wanted working, so I probably just use the Denon Ceol's interface for browsing. It's clunky and hard to use but workable.

**Some minor issues I had**

The mount would disappear after reboot; turns out I messed up /etc/fstab. Easy fix.
Our Denon Ceol wasn't connected to the network; I had to create a separate network for it as I refused to connect it to our main or IoT networks for some reason. I will work on this more.
The HEOS app on my Samsung also doesn't pick up the Ceol sometimes; a quick app close normally fixes this. Sometimes I have to delete the user data for the app and relogin to get it to work. 


**A Few Quick Tips**
EasyTAG has the ability to edit the tags of multiple files in the same folder at once; just click on the little T+ symbol on the right side of the text input field. Make sure the song you want to edit is highlighted first.
Since Jellyfin's DLNA server no longer allows anything but the predefined tags to be browseable, this leaves Classical music owners in the mud, as composeris not surfaced. I used Album Artist instead; this leaves Artist to be used for the Performers. Check beforehand though, apparently Jellyfin used to be able to surface any tag for broswing, but this was changed, it may change back. 

**My thought so far**

Overall, I'm relatively happy with this setup so far. The DLNA server seems to be quite reliable on the Pixel phone at least. My main complaint is how inconsistent the music databases are with their tags. To be fair, I don't think they're "wrong" as such — the way someone tags their music is really just down to personal preference, and there's no objectively correct way to do it. The trouble is that everyone's preferences are different, so what I pull down from one database rarely matches what I pulled from another, and neither necessarily matches how I'd do it myself. If they were all consistent with each other I wouldn't mind nearly as much, but each one does things its own way — which I suppose is exactly what you'd expect from user-generated databases
I think I might change Fre:ac's file structure from <artist>/<album>/<track>-<title> to <album>/<artist>/<track>-<title>, as some tracks from albums with different pieces performed by different performers end up in different folders if the person labbeling them put didn't combine artists, which I find annoying.


