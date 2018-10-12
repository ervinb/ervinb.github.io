---
layout: article
title: Troubleshooting Gigabyte GA-970A-DS3P BIOS freeze
tags: bios troubleshooting freeze hangup bios setup
key: bios-freeze
---

This one started out as a nice morning surprise. A bad habit of mine is
to turn on the computer in the morning, almost while I'm stretching after waking up.

Not much later, I was greeted with the following message:

```
Reboot and Select proper Boot device or Insert Boot Media in selected Boot device and press a key
```

Naturally, I thought that my SSD drive gave up on me, so I did a reboot and tried
to select my mechanical drive with Fedora 25 on it. As soon as the motherboard's
boot menu showed up, everything froze. Strange. Did another reboot to enter the BIOS
setup. Again, once the setup screen (this time, partially) loaded up, everything
froze. No input was possible from the keyboard or the mouse.

## Swim!

The "*my left leg is inside a shark's mouth*" version:

**Disconnect all your USB peripherals, starting with game controllers.**


## Finding Controlly

I had similar instances with other PC's, where hard drives were playing up,
and the BIOS refused to POST. This was usually indicated with a constantly
glowing HDD activity light. Now, the symptoms weren't exactly the same, but I decided
to disconnect all the drives and give it a go.

Master power off, case out, side down, SATA cables out, reboot. Nothing.
Clear the CMOS, do a power drain. Nothing. Everything froze at the setup screen.
Load up the harpoon.

So, the drives were fine, the PC POSTed but couldn't load the setup nor the boot menu,
and it spat out the error message above, once it got to loading the OS. A fishy smell
filled the room.

Started pulling out all the connected peripherals, leaving only the keyboard
and the mouse. Bam! It worked!

It turns out, the connected PS4 controller was causing the BIOS to hang and broke the
boot process as well. The night before, I connected my PS4 controller via USB to charge it up, and
left it on the cable. The moment I disconnected it, everything was working again. Credit
goes to `ndisic` from [Gigabyte forums](http://forum.giga-byte.co.uk/index.php?PHPSESSID=6kfiua6jibl4ntdptauks3fu91&topic=14493.0)
for the hint.

## Lying on the shore

This is an edge case and the freezups can be caused by bad memory sticks or
failing drives. The error message above indicates a failing drive in 99% of the cases.
However, it's interesting to see these exotic instances from time to time.

Put the harpoon aside until next time.
