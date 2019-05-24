---
layout: post
title:  "PXE-booting IncludeOS with Grub"
author: perbu
published: true
date:   2019-03-27 12:43:42 +0200
categories: [development,tooling,boot,bare metal]
hero: /assets/img/posts/boots.jpg
author-image: /assets/img/authors/perbu.jpg
summary: "Howto boot IncludeOS with Grub and PXE. Could be useful for booting Linux as well."
---

PXE, pre-execution environment, provides a simple, old, slightly clunky, but working way of booting computers of the network. I’ve previously used it for provisioning Linux servers and desktops, automating the tedious task of clicking through Linux installers.

So far, as IncludeOS has been focused on virtual machines and as such testing, it has been a matter of just invoking Qemu or whatever hypervisor we wanted on the image, so there hasn’t been a good reason to look into booting off the network.

This week we needed to document some behavior when running on bare metal. However, something in the IncludeOS hardware abstraction layer wasn’t compatible with hardware in the machine we wanted to experiment with. This led to panic during boot. Figuring these things out require iteration and iterating back and forth across the floor with a memory stick is tedious.

So, when we learned that our boot-loader of choice, Grub (previously known as Grub2), supports PXE, we immediately try to set this up. There are good reasons to use Grub, even if you’re want to boot Linux as invoking Grub over the network allows you to use the Grub shell to inspect and boot of arbitrary partitions on your local hard drives.

Here is what we did to get Grub to boot over the network on Ubuntu 18.04.

First, we disabled the built-in DHCP server in our router and installed “isc-dhcp-server” on the Linux server. You might be able to reconfigure your DHCP server to support PXE if your DHCP server is less braindead the stuff we currently have.

Your ```dhcpd.conf``` should look something like this:
```
option domain-name "includeos.org"; # you might wanna change this.
option domain-name-servers 1.1.1.1, 8.8.8.8, 8.8.4.4; # Cloudflare, Google

default-lease-time 600;
max-lease-time 7200;
allow bootp;            # allow BOOTP
ddns-update-style

subnet 192.168.1.0 netmask 255.255.255.0 {
   range 192.168.1.40 192.168.1.250;
   option routers 192.168.1.1;
   range dynamic-bootp 192.168.1.30 192.168.1.39;

   class "pxeclients" {
                 match if substring (option vendor-class-identifier, 0, 9) = "PXEClient";
                 option tftp-server-name "192.168.1.2";   # our IP address
                 option bootfile-name   "/boot/grub/i386-pc/core.0";
   }
}
```
So now dhcpd send some extra information along with the IP leases so that clients can boot. Next, we need to have a way to transfer binaries to the clients. Here tftp, the trivial file transfer protocol, is used. It’s a simple and insecure protocol to transfer files. ```apt install tftpd-hpa``` will install a suitable server. It starts up right away and will serve files out of ```/var/lib/tftpboot```. So we go there. Grub comes with a script to help set up everything for netbooting, grub-mknetdir.
```
$ grub-mknetdir --net-directory=/var/lib/tftpboot/
```
If you look at the structure, you’ll find the Grub core that was referred to in the dhcpd.conf. This will load and you’ll find the familiar Grub prompt appearing. To load IncludeOS you’ll use the “multiboot” keyword. So if I put an IncludeOS instance, “seed”, in “/var/lib/tftpboot” I can then issue “multiboot seed” in Grub and grub will download the instance and load it.
“boot” will execute it and load it.

It is easy to add a menu listing a few choices. The Grub core will try to load “grub.cfg” from boot/grub. boot/grub is a relative path relative to the root of the tftp server so the full path is ```/var/lib/tftpboot/boot/grub```. Here is our current grub.cfg:
```
menuentry "IncludeOS hello" {
      multiboot /hello
}

menuentry "IncludeOS seed" {
      multiboot /seed
}
```
The files referenced in the Grub configuration are placed in the root of the tftp server,  ```/var/lib/tftpboot/boot/grub```.

In addition to PXE-booting IncludeOS, I would also recommend getting a serial console if you are going to work on the operating system itself. Serial ports are so primitive that they will work almost no matter what. So a sure-fire way of getting a stack trace is to have a serial console connected to the machine you’re working on. We use a microcontroller-based serial-to-telnet gateway that allows us to connect netcat to it and dump the serial console automatically into a log file. So even if the stack trace is three pages long we don't have to resolve to filming the screen in slow motion in order to see what is going on. 

