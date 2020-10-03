---
layout: post
title:  "Docker & ZFS On Debian"
date:   2020-10-03 9:45:00 -0500
categories: homelab
---

*Building a capable Docker Machine on a Dell R710*

{:refdef: style="text-align: center;"}
![OpenZFS, Docker, Debian Logos](/assets/img/zfs_docker_debian.png)
{: refdef}

# How'd we Get Here...
A while back, I snagged a Dell R710 from Ebay for $350 delivered. This thing is a virtualization *monster*. 8 cores and 144GB of ECC RAM, perfect for filling with bhyve virtual machines on FreeNAS. This setup worked great for me until I became interested in Docker containers. FreeNAS just doesn't support a docker environment, so let's build something from the ground up and document the process.

# Constraints
- Must use ZFS (no need for ZFS root)
- Native docker support
- Lightweight (no GUI, unnecessary package systems \*cough cough snaps\*, etc.)

With these constraints, we know we absolutely must use Linux and we would really like a distro that has openzfs in its package control. In the past I've used Ubuntu Server and ubuntu 16.04 has zfs available as a package, but I want to avoid snaps like the plague. Because of this, I'm going to use Debian.

# Initial Setup
First things first, we're going to shut down the server and remove every drive but our desired boot drive. FreeNAS uses a flash drive as its root storage, but debian doesn't quite support that. In my case, I'm going to use an old 120GB SSD in an optical drive caddy in place of the server's disk drive. It's easy enough to complete the guided install, and Debian has some pretty good installation literature:
- [Installation Guide - Long](https://www.debian.org/releases/stable/i386/index.en.html)
- [Installation Howto - Short](https://www.debian.org/releases/stable/i386/apa.en.html)

I do have a few gotchas during the guided install:
- Some NICs require closed-source firmware to operate. The NIC in my R710 is one of those, so I had to use the [non-free debian image](https://cdimage.debian.org/cdimage/unofficial/non-free/cd-including-firmware/current/).
- Guided partitioning failed for my install. I suspect that this is because it was trying to make a gigantic swap partition to match the 144GB of RAM. I decided not to use a swap partition and partition the entire SSD as `/`.
- I avoided giving root a password. Instead, I have an administrator user with sudo access.
- I did not install any desktop environments or X server components.
- Don't forget to select openssh!

# Post-Install
Now that we have a fully running debian system, let's change a few things. Pop all your drives back in and ssh in.

Let's gear up to use ssh without a password, starting by setting up private key authentication. If you're on linux, you've got it easy: `$ ssh-copy-id $USER@$HOST`. On Windows, I find it easiest to manually copy the contents of `%HOMEPATH%\.ssh\id_rsa.pub` to `~/.ssh/authorized_keys` on my server. If you don't have an `~/.ssh/id_rsa.pub` or `%HOMEPATH%\.ssh\id_rsa.pub`, you better generate an ssh key with `$ ssh-keygen`.

Once your public key is copied in, it's a good idea to exit your ssh session and attempt to login with private key auth. If you don't need a password, you're on the right path. Time to disable password authentication. In `/etc/ssh/sshd_config`, find the lines containing the following keys, uncomment them and make sure they're set to `no`.

```
PasswordAuthentication no
ChallengeResponseAuthentication no
PermitRootLogin no
UsePAM no
```

Now you can reload sshd and bask in paswordless security strong enough to leave exposed publicly. `$ sudo /etc/init.d/sshd reload`

# ZFS Time
Debian makes this one super easy. Check the [official ZFS Debian Wiki Page](https://wiki.debian.org/ZFS) for more info. Use the following commands to install ZFS on Debian.
```
$ sudo apt update
$ sudo apt install linux-headers-`uname -r`
$ sudo apt install -t buster-backports dkms sol-dkms
$ sudo apt install -t buster-backports zfs-dkms zfsutils-linux
```
It may take a little while to run the dkms build, and it may throw a few warnings. They do not specifically recommend a reboot on the Debian Wiki, but it's never a bad idea when dealing with kernel modules.

Just for grins, if you happen to be installing on a system that used to run FreeNAS, you can try to import your old pool with `$ sudo zpool import`. You're likely to see `action: The pool can be imported using its name or numeric identifier and the '-f' flag.`. You can force the import by running `$ sudo zpool import -f $POOLNAME`.

If you're starting fresh, now is the time to make your first zfs pool. I'm in a bit of a hard drive shortage, so for testing's sake I will be doing a stripe pool with two 500GB hard drives. ZFS uses disk IDs or paths to reference disks instead of /dev/sdX paths. You can figure out which disk is which with the following commands:
```
$ ls -l /dev/disk/by-id
$ ls -l /dev/disk/by-path
$ lsblk -o NAME,SIZE,MODEL,VENDOR
```
With the device IDs in hand, I'll make a zpool with:
```
$ sudo zpool create tank ata-ST9500325AS_6VESPM0A ata-ST9500325AS_6VESRNXY
```

# Docker Install
This is essentially a summary of the [Docker Docs for Debian](https://docs.docker.com/engine/install/debian/).

1. Update apt and install required packages:
```
$ sudo apt update
$ sudo apt install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
```
2. Get Docker's GPG key:
```
$ curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -
```
You can verify you have the correct key as follows:
```
$ sudo apt-key fingerprint 0EBFCD88
pub   4096R/0EBFCD88 2017-02-22
      Key fingerprint = 9DC8 5822 9FC7 DD38 854A  E2D8 8D81 803C 0EBF CD88
uid                  Docker Release (CE deb) <docker@docker.com>
sub   4096R/F273FCD8 2017-02-22
```
3. Use the following command to set up Docker stable.
```
$ sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/debian \
   $(lsb_release -cs) \
   stable"
```
4. Finally, install docker!
```
$ sudo apt update
$ sudo apt install docker-ce docker-ce-cli containerd-io
```
5. If you want to allow other users to run docker commands, you can set that up [as follows](https://docs.docker.com/engine/install/linux-postinstall/):
```
$ sudo groupadd docker
$ sudo usermod -aG docker $USER
```
Log out and back in
1. You can verify that your docker install is functional by running `hello-world`
```
$ docker run --rm hello-world
```

# Let's Hook up Docker and ZFS
This part's for the adventurous. You could certainly just make a bunch of ZFS datasets with filesystem mounts and mount those in your containers for bulk storage, but that's no fun! Docker has a [ZFS storage driver](https://docs.docker.com/storage/storagedriver/zfs-driver/) that will back all container, image, and volume storage with zfs. Let's set it up.
1. Kill docker:
```
$ sudo /etc/init.d/docker stop
```
2. Back up /var/lib/docker just in case and then delete its contents:
```
$ sudo cp -au /var/lib/docker /var/lib/docker.bk
$ sudo rm -rf /var/lib/docker/*
```
3. Create a ZFS-backed mountpoint at `/var/lib/docker`. The official docker instructions suggest making a zpool and mounting it there, but I'll use a dataset since I already have a zpool configured (name `tank`).
```
$ sudo zfs create -o mountpoint=/var/lib/docker tank/docker
```
4. Tell Docker to use ZFS for its storage driver. In `/etc/docker/daemon.json`:
```
{
  "storage-driver": "zfs"
}
```
5. Start docker back up and verify you're using the ZFS storage driver:
```
$ sudo /etc/init.d/docker start
$ docker info
Server:
 Containers: 0
  Running: 0
  Paused: 0
  Stopped: 0
 Images: 114
 Server Version: 19.03.11
 Storage Driver: zfs
  Zpool: tank
  Zpool Health: ONLINE
  Parent Dataset: tank/docker
  Space Used By Parent: 17283463680
  Space Available: 935802788864
  Parent Quota: no
  Compression: off
 Logging Driver: json-file
 Cgroup Driver: cgroupfs
```

# 🎉 Done!
Now we have a server running Debian with Docker using ZFS-backed storage. This would be a great way to put together a NAS setup using samba or NFS with a bunch of self-hosted services. I'll be using Docker to host a home assistant instance with nginx sitting in front as a reverse proxy.