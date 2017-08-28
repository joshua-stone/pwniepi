# Overview

This project is an attempt to learn penetration testing using a suite of open source tools, with as few modifications as possible along with a low barrier of entry.

The Raspberry Pi is an appealing testbed for pentesting due to its low cost, great community support, small form factor, and simple design. Open source support also appears to have matured to the point that many distros now provide upstream builds that're compatible with the Raspberry Pi.

I've chosen Fedora for this project because it uses mainline Linux kernels that're kept up to date. This is especially important for a platform like the Raspberry Pi which benefits from hardware support and enablement that's added in newer kernels.

## Hardware Prerequisites

* [Raspberry Pi 3 model B](https://www.raspberrypi.org/products/raspberry-pi-3-model-b/#buy-now-modal)

* 2.5A microUSB power supply

* microSD card

## Installing Fedora

### Step 1: Downloading installer

As of Fedora 25, there are official ARM builds that support the Raspberry Pi. First download [one of the images from the Fedora site](https://mirrors.syringanetworks.net/fedora/linux/releases/26/Spins/armhfp/images/).

```bash
wget https://mirrors.syringanetworks.net/fedora/linux/releases/26/Spins/armhfp/images/Fedora-Minimal-armhfp-26-1.5-sda.raw.xz
```

### Step 2: Verifying installer

Now that the image has been downloaded, be sure to verify that it's an offical image and not one that's been altered ([looking at you, Mint](http://blog.linuxmint.com/?p=2994)).

```bash
gpg --keyserver hkp://keys.gnupg.net --recv-key 64DAB85D
wget https://mirrors.syringanetworks.net/fedora/linux/releases/26/Spins/armhfp/images/Fedora-Spins-26-1.5-armhfp-CHECKSUM
gpg --verify Fedora-Spins-26-1.5-armhfp-CHECKSUM
sha256sum --check --ignore-missing Fedora-Spins-26-1.5-armhfp-CHECKSUM 
```


