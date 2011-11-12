---
layout: post
title: FAI Lab With VirtualBox (Part III)
---

<small>*Note: This post is the third in a series of posts about
 setting up an FAI lab using VirtualBox. Here are [part I][1] and
 [part II][2]*</small>

In [Part I][1] of this series we set up a [VirtualBox][3] virtual
machine to act as a DHCP/PXE server and configured a basic [FAI][4]
nfsroot for other VMs to boot into. In [Part II][2] we configured FAI
so that we could install Ubuntu onto PXE-booted virtual machines. In
this part, I was going to tell you how to FAI a Redhat system, then
see if we could make it easy to PXE boot into a "live" system for
debugging, etc. Then I talked to a colleague who is more familiar with
RPM-based distributions and he assured me that FAI was the wrong way
to do it, and told me to use kickstart. After some basic research, I'm
definitely inclined to agree. So instead of using FAI to bootstrap our
Redhat VMS, I'm going to show you how to PXE-boot off our existing
"faiserver" VM into a kickstart installation.

### Kickstarting a Redhat install

By now you should be familiar with the VirtualBox commands to create a
new VM configured to network boot. Refer to [part II][2] if you need a
refresher . Create a VM (don't forget to use the correct `--ostype`
argument); I'm calling mine "redhat" for the purposes of this
article. Add the MAC to `/etc/ethers` on "faiserver" and assign it an
IP address in `/etc/hosts`. Use `pkill -HUP dnsmasq` to get dnsmasq to
reload those files.

Now we need to set up our kickstart environment. You'll need an ISO of
the Redhat version you want to install. Create `/srv/kickstart` (or
whatever path you want) and `/srv/kickstart/iso`. Mount the ISO
(`mount -oloop /srv/kickstart/rhel-server-5.4-x86_64-dvd.iso
/srv/kickstart/iso`, and you may want to add it to `/etc/fstab`) at
`/srv/kickstart/iso` and add it to `/etc/exports`. Put your kickstart
file in `/srv/kickstart` (more on getting the kickstart file later)
and export that as well. I called mine
`/srv/kickstart/rhel5.4-x86_64.cfg`. There are a few items we need to
copy off of the ISO into the TFTP server directory. Copy
`/srv/kickstart/iso/images/pxeboot/vmlinuz` to
`/srv/tftp/fai/vmlinuz-rhel5.4-x86_64` and
`/srv/kickstart/iso/images/pxeboot/initrd.img` to
`/srv/tftp/fai/initrd.img-rhel5.4-x86_64`. (We're hijacking the "fai"
sub-directory of the TFTP server, but that's not really important.)

Next, we need to create a file in `/srv/tftp/fai/pxelinux.cfg` to tell
the PXE-booting system which kernel, initrd, and kernel parameters to
use. We don't have the convenient `fai-chboot` command to generate
this file for us in the case of Redhat (though you could use it to
make a template if you'd like). The filename should be the IP address
of the host you are booting, in hex. My "redhat" VM has the IP address
192.168.0.11 - I could leave this as an exercise to the reader, but
I'm always forgetting how to do this too, so:

   1. Remember, each component of the IP address is 2 hexadecimal
      digits.
   1. The first component is 192. `printf '%x\n' 192` gives us `c0`.
   1. `printf '%x\n' 168` gives us `a8`.
   1. `0` in hex is just `0`, but each component is to digits, so:
      `00`.
   1. Finally, `11` in hex is easy to remember - that's just
      `b`. Again, two digits, so `0b`.
   1. The filename is in all caps, so we are left with: `C0A8000B` as
      the filename we need to create, with the following contents.

{% highlight text linenos %}
default kickstarter

label kickstarter
kernel vmlinuz-rhel5.4-x86_64
append initrd=initrd.img-rhel5.4-x86_64 ip=dhcp autostep ks=nfs:192.168.0.2:/srv/kickstart/rhel5.4-x86_64-ks.cfg method=nfs:192.168.0.2:/srv/kickstart/iso text
{% endhighlight %}

   1. This line tells the boot loader to load the section labeled
      "kickstarter".
   1. Blank.
   1. This is the label referred to by line #1.
   1. This is the name of the kernel we're going to boot (which we
      copied from the RHEL ISO).
   1. These are the kernel parameters.
      * `initrd=initrd.img-rhel5.4-x86_64` specifies the initrd we
       copied from the RHEL ISO.
      * `ip=dhcp` tells the kernel to use DHCP to configure the
        network interface.
      * `autostep` tells kickstart to run non-interactively.
      * `ks=nfs:192.168.0.2:/srv/kickstart/rhel5.4-x86_64-ks.cfg` tells
        kickstart where to locate the kickstart file.
      * `method=nfs:192.168.0.2:/srv/kickstart/iso` tells kickstart
        where to locate the RHEL installation tree (the ISO we
        mounted and exported).
      * `text` tells kickstart to run in text mode.

I dug these kernel parameters out of the
[Red Hat Enterprise Linux 5 Installation Guide][5]; specifically
[section 31.10][6].

"Where did this kickstart file come from?" you are wondering. Well,
the easiest way to get one is to run through the install process
manually once. This will leave a kickstart configuration in
`/root/anaconda-ks.cfg`. You can then tweak this file until you have a
solid basic installation to work from. [Here's mine][7] - the
[documentation][8] is actually pretty good, and will help you modify
the basic kickstart file to match your needs.

Now that we have a way to bootstrap multiple distributions, we're in
pretty good shape for our FAI lab. I'm going to take some time to
implement what I've learned in production. Next time I write I'll try
to show you how to PXE boot into a "live" system for debugging, etc.

[1]: /2011/11/01/fai-lab-with-virtualbox-part-i.html
[2]: /2011/11/10/fai-lab-with-virtualbox-part-ii.html
[3]: https://www.virtualbox.org/
[4]: http://fai-project.org/
[5]: http://docs.redhat.com/docs/en-US/Red_Hat_Enterprise_Linux/5/html/Installation_Guide/index.html
[6]: http://docs.redhat.com/docs/en-US/Red_Hat_Enterprise_Linux/5/html/Installation_Guide/s1-kickstart2-startinginstall.html
[7]: /files/failab/rhel5.4-x86_64-ks.cfg
[8]: http://docs.redhat.com/docs/en-US/Red_Hat_Enterprise_Linux/5/html/Installation_Guide/s1-kickstart2-options.html
