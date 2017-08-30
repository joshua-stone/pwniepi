# Overview

This project is an attempt to learn penetration testing using a suite of open source tools.

The Raspberry Pi is an appealing testbed for pentesting due to its low cost, great community support, small form factor, and simple design. Open source support also appears to have matured to the point that many distros now provide upstream builds that're compatible with the Raspberry Pi.

Fedora was chosen for this project because it uses mainline Linux kernels that're kept up to date. This is especially important for a platform like the Raspberry Pi which benefits from hardware support and enablement that's added in newer kernels. This project also depends on Fedora-specific tools (portability is important, but so is development time), including `dnf` and `fedora-arm-installer`.

## Hardware Prerequisites

* [Raspberry Pi 3 model B](https://www.raspberrypi.org/products/raspberry-pi-3-model-b/#buy-now-modal)

* 2.5A microUSB power supply

* >=16GB microSD

* HDMI monitor w/ cable

* USB keyboard

## Installing Fedora

### Step 1: Downloading installer

As of Fedora 26, there are official ARM builds that support the Raspberry Pi. First download [one of the images from the Fedora site](https://mirrors.syringanetworks.net/fedora/linux/releases/26/Spins/armhfp/images/).

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

### Step 3: Generating SSH keys

Having strong SSH keys is of utmost important for secure system access. Newer versions of ssh-keygen can use [Ed25519](https://ed25519.cr.yp.to/) which generates keys that have a complexity similar to 4096-bit RSA.

```bash
ssh-keygen -t ed25519 -a 100
```

If backwards compatibility with older systems is important, then RSA will work fine.

```bash
ssh-keygen -t rsa -b 4096 -o -a 100
```

### Step 4: Installing Fedora

The easiest way to install Fedora onto a microSD card is with the fedora-arm-installer utility. The target MicroSD card should be available as `/dev/mmcblk0` and NOT mounted, which can be verified by running `lsblk`. 

Be sure to verify that all filenames are correct before flashing the microSD card.

```bash
sudo arm-image-installer --image=Fedora-Minimal-armhfp-26-1.5-sda.raw.xz \
--addkey="${HOME}/.ssh/id_ed25519.pub" \
--target=rpi3 \
--resizefs \
--media=/dev/mmcblk0
```

### Step 5: Enabling Wi-fi

Fedora currently doesn't enable RPi3's wifi chip out of the box unless copying a specific text file.

```bash
sudo mkdir /mnt/pwniepi
sudo mount /dev/mmcblk0p4 /mnt/pwniepi
sudo cp configs/lib/firmware/brcm/brcmfmac43430-sdio.txt /mnt/pwniepi/lib/firmware/brcm/brcmfmac43430-sdio.txt
```

### Step 6: Bootstrapping pentesting suite

Fedora's DNF package manager supports custom install roots and specifying architectures. DNF also supports package groups, the most relevant being [security-lab](https://github.com/fabaff/fsl-test-bench/blob/master/fsl.yml). In this case, all relevant packages can be downloaded to DNF's cache stored on the microSD card.

```bash
sudo dnf --downloadonly --forcearch=armv7hl --installroot /mnt/pwniepi install @security-lab alien bzip2 gnutls-utils mingw32-nsis openvas-manager openvas-gsa redis tar texlive-collection-latexextra sqlite
sudo dnf --downloadonly --forcearch=armv7hl --installroot /mnt/pwniepi update
```

Once finished, the microSD card can now be removed.

```bash
sudo umount /mnt/pwniepi
sync
```

### Step 7: Booting the Raspberry Pi

The microSD card should be ready to boot at this point, so insert it into the RPi3, connect an HDMI monitor and USB mouse, and plug in a power supply. In a moment there'll be a menu to configure root password, timezone, and user. Once everything is configured, a login shell should be available.

Currently there's a bug where scripts fail when installing packages onto the mounted microSD card on a host system, so booting the Raspberry Pi to complete the install process is the safest method.

```bash
sudo dnf install /var/cache/dnf/{fedora,updates}-*/packages/*.rpm
sudo dnf clean packages
```

SSH keys should also be copied from `/root` so non-root users can log in via SSH

```bash
sudo cp -r /root/.ssh ~
chown -R ${USER}:${USER} ~/.ssh
```

Disabling password authentication and root login should now be safe, which should appear like so in `/etc/ssh/sshd_config`:

```
PubkeyAuthentication yes
PasswordAuthentication no
PermitRootLogin no
```

Now reload SSH.

```bash
sudo systemctl reload sshd
```

Finally, disable SELinux so that OpenVAS can be configured, which should appear like so in `/etc/selinux/config`:

```
SELINUX=disabled
```

Now reboot.

```bash
sudo reboot
```

## Configuring OpenVAS

### Step 1: Set up Redis

Redis must be listening on a Unix socket, so configure `/etc/redis.conf` to have these parameters:

```
port 0
unixsocket /tmp/redis.sock
unixsocketperm 700
```

Redis should be ready to start.

```bash
sudo systemctl enable redis
sudo systemctl start redis
```

### Step 2: Update OpenVAS network vulnerability tests

Ensure that OpenVAS keeps an up-to-date collection of NVTs.

```bash
sudo greenbone-nvt-sync
```

NVT signature checking isn't enabled by default, so enable it in `/etc/openvas/openvassd.conf`:

```
nasl_no_signature_check = no
```

```bash
sudo mkdir -p /var/lib/openvas/openvasmd/gnupg
sudo gpg --keyserver hkp://keys.gnupg.net --homedir=/etc/openvas/gnupg --recv-key 48DB4530
```

```bash
sudo greenbone-certdata-sync
sudo greenbone-scapdata-sync
```

```bash
sudo openvassd
sudo openvasmd --rebuild
```

### Step 3: Create an OpenVAS user

OpenVAS needs an admin user, but first there should be a password policy set in `/etc/openvas/pwpolicy.conf` because there aren't any password restrictions set by default. Fortunately there are premade policy rules that can be uncommented:

```bash
# +desc: Too short (at least 8 characters are required)
!/^.{8,}$/
```

Once a password policy is set, create an admin user.

```bash
sudo openvasmd --create-user=<name> --role=Admin
sudo openvasmd --user=<name> --new-password=<password>
```

### Step 5: Starting the OpenVAS manager and assistant

The OpenVAS Manager is a core component that acts as a layer between OpenVAS Scanner and clients, while Greenbone Security Assistant provides a web interface that connects to OpenVAS Manager.

```bash
sudo openvas-manage-certs -a
sudo openvasmd
sudo gsad
```

### Step 4: Checking OpenVAS setup

There should be a working OpenVAS setup at this point.

```bash
sudo openvas-check-setup --v9
```

If everything looks good, then there should be an admin interface accessible on [http://127.0.0.1](http://127.0.0.1/)
