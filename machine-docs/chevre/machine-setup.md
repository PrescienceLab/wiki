# Chevre
Chevre (`chevre.cs.northwestern.edu`) is a Cheri-enabled Arm Morello software development machine.

# Useful Links
  * https://www.cheribsd.org/
  * https://ctsrd-cheri.github.io/cheribsd-getting-started/cover/index.html
  * https://developer.arm.com/documentation/den0132/0100

# Fresh Setup
A fresh setup involves doing everything, including installing and/or upgrading the motherboard firmware and installing the operating system.

## Installing and/or Upgrading Motherboard Firmware
There are two ways to install/update the motherboard firmware.
  1. Flash the firmware SD card with a new firmware from another computer.
  2. Flash the firmware SD card over a serial connection to the "guest" Morello computer.

Exact details are covered in the documentation on [Upgrading the Morello firmware](https://ctsrd-cheri.github.io/cheribsd-getting-started/morello-firmware/index.html).

## Installing the Operating System
Throughout this setup we will assume you are installing [CheriBSD](https://www.cheribsd.org/) [CheriBSD](https://www.cheribsd.org/) to your Morello setup.
This is a pretty normal setup process, except you will not have HDMI output.
You will need to do all of your installation from the serial console instead.

> **NOTE**: To get any output on the HDMI output, you must install a desktop environment on CheriBSD!
> Without this, all interaction will be done over the serial port on `MCC PORT`.

Once you enter the live image's shell, you are in a normal FreeBSD system.
  1. Confirm you are booted from the USB stick with `camcontrol devlist`.
  2. Use `bsdinstall` to run the TUI installer for FreeBSD/CheriBSD.
  3. You may need to create a `tmpfs`, if the live installer did not provide one automatically.
     This can be done with `mount -t tmpfs tmpfs /tmp`.
     NOTE: The `-t tmpfs` says what type of mount this is, and the second `tmpfs` says the "source device" for the filesystem.
