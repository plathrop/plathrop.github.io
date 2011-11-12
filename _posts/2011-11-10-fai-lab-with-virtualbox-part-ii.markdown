---
layout: post
title: FAI Lab With VirtualBox (Part II)
---

<small>*Note: This post is the second in a series of posts about setting up
 an FAI lab using VirtualBox. The first post can be found [here][1].*</small>

In [Part I][1] of this series we configured a [VirtualBox][2] virtual
machine with a basic installation of [FAI][3]. We also configured a
virtual network and connected our FAI server to it. Finally, we set up
a DHCP/TFTP server to allow us to PXE boot clients into the FAI
nfsroot. The next step is to boot up our first FAI client virtual
machine to make sure the PXE process works correctly. Then, we'll
configure FAI to install and configure Ubuntu Server on our client.

### Testing PXE Booting

Create a new VM to test PXE booting. These commands will be familiar
from Part I:

{% highlight bash linenos %}
VBoxManage createvm --name pxetest --ostype Ubuntu_64 --register
VBoxManage modifyvm pxetest --memory 512 --boot1 disk --boot2 dvd --boot3 net --rtcuseutc on
VBoxManage modifyvm pxetest --nic1 hostonly --hostonlyadapter1 vboxnet0
VBoxManage createhd --size 10000 --filename $HOME/Library/VirtualBox/HardDisks/pxetest-root
VBoxManage storagectl pxetest --name sata0 --add sata
VBoxManage storageattach pxetest --storagectl sata0 --device 0 --port 0 --type hdd --medium $HOME/Library/VirtualBox/HardDisks/pxetest-root.vdi
{% endhighlight %}

We're going to need to know the MAC address of the host in order to
configure dnsmasq to PXE boot it.

{% highlight text %}
plathrop ~ » VBoxManage showvminfo pxetest | grep MAC
NIC 1:           MAC: 080027928409, Attachment: Host-only Interface 'vboxnet0', Cable connected: on, Trace: off (file: none), Type: 82540EM, Reported speed: 0 Mbps, Boot priority: 0, Promisc Policy: deny
{% endhighlight %}

As you can see, the MAC for the "pxetest" VM is
`08:00:27:92:84:09`. Add the MAC to `/etc/ethers` on the "faiserver"
VM, along with the hostname you want this system to use.

{% highlight text %}
08:00:27:92:84:09	pxetest
{% endhighlight %}

You'll need to add a record to `/etc/hosts` as well, mapping the name
(pxetest) to an IP address from the range we configured dnsmasq to
pass out (192.168.0.10 onward):

{% highlight text %}
192.168.0.10    pxetest
{% endhighlight %}

Restart dnsmasq on the "faiserver" VM and boot your "pxetest" VM. You
should see something like this in the VirtualBox window:

![pxetest boot screenshot](/images/failab/pxetest.png)

Despite the error message, this means we've succeeded! The "pxetest"
VM received an IP address from dnsmasq, and the required options to
PXE boot. If you look at your dnsmasq logs (in `/var/log/daemon.log`
on Ubuntu) you'll see something like this:

{% highlight text linenos %}
Nov  8 00:30:59 faiserver dnsmasq-tftp[5065]: sent /srv/tftp/fai/pxelinux.0 to 192.168.0.10
Nov  8 00:30:59 faiserver dnsmasq-tftp[5065]: file /srv/tftp/fai/pxelinux.cfg/2b176787-c273-4728-a8db-7af779beb2b3 not found
Nov  8 00:30:59 faiserver dnsmasq-tftp[5065]: file /srv/tftp/fai/pxelinux.cfg/01-08-00-27-92-84-09 not found
Nov  8 00:30:59 faiserver dnsmasq-tftp[5065]: file /srv/tftp/fai/pxelinux.cfg/C0A8000A not found
Nov  8 00:30:59 faiserver dnsmasq-tftp[5065]: file /srv/tftp/fai/pxelinux.cfg/C0A8000 not found
Nov  8 00:30:59 faiserver dnsmasq-tftp[5065]: file /srv/tftp/fai/pxelinux.cfg/C0A800 not found
Nov  8 00:30:59 faiserver dnsmasq-tftp[5065]: file /srv/tftp/fai/pxelinux.cfg/C0A80 not found
Nov  8 00:30:59 faiserver dnsmasq-tftp[5065]: file /srv/tftp/fai/pxelinux.cfg/C0A8 not found
Nov  8 00:30:59 faiserver dnsmasq-tftp[5065]: file /srv/tftp/fai/pxelinux.cfg/C0A not found
Nov  8 00:30:59 faiserver dnsmasq-tftp[5065]: file /srv/tftp/fai/pxelinux.cfg/C0 not found
Nov  8 00:30:59 faiserver dnsmasq-tftp[5065]: file /srv/tftp/fai/pxelinux.cfg/C not found
Nov  8 00:30:59 faiserver dnsmasq-tftp[5065]: file /srv/tftp/fai/pxelinux.cfg/default not found
{% endhighlight %}

As we can see, the PXE boot failed because no PXE config was located
for the "pxetest" VM. Let's fix that. The command we use is
[fai-chboot(8)][4], like this:

{% highlight text linenos %}
faiserver ~ » sudo fai-chboot -I -F -v 192.168.0.10
Booting kernel vmlinuz-2.6.38-grml
 append initrd=initrd.img-2.6.38-grml ip=dhcp
    FAI_FLAGS=verbose,sshd,createvt

192.168.0.10 has 192.168.0.10 in hex C0A8000A
Writing file /srv/tftp/fai/pxelinux.cfg/C0A8000A for 192.168.0.10
{% endhighlight %}

<small> *Note: you can ignore the -grml stuff; I'm using a custom
kernel and initrd in my environment, it should not affect the rest of
this article.* </small>

The `-F` option sets some useful flags for FAI, `-I` tells FAI to run
an automatic installation, and `-v` gives us verbose output. The IP
address is the IP we assigned to the "pxetest" VM. As you can see in
line 7, this command creates the file
`/srv/tftp/fai/pxelinux.cfg/C0A8000A`, which is one of the files that
the TFTP server was looking for (see line 4 of the previous
listing). Now that this is in place, if you reboot the "pxetest" VM,
you should see the system PXE boot, mount the nfsroot, and then dump
you out to a prompt when FAI fails (because we haven't created an FAI
configuration yet). If the machine PXE boots fine, but doesn't get the
correct IP address from DHCP, you probably did not disable the
VirtualBox DHCP server on `vboxnet0` as described in "Setting up the
FAI server VM" in [Part I][1].

### Installing Ubuntu via FAI

Now we need to configure FAI to actually perform an installation. To
start with we'll need to copy the example configuration provided by
FAI. This will give us a simple FAI configuration to build on top of.

{% highlight text %}
sudo cp -a /usr/share/doc/fai-doc/examples/simple/* /srv/fai/config
{% endhighlight %}

We'll also need to NFS export our configuration directory, so we add
it to `/etc/exports`:

{% highlight text %}
/srv/fai/config  192.168.0.2/16(async,ro,no_subtree_check,no_root_squash)
{% endhighlight %}

Activate this export with `sudo exportfs -a`. We also need to do one
last bit of network configuration to allow our FAI clients to speak to
the outside world during the install process. We'll need to make sure
the "faiserver" VM is configured as a NAT gateway. Add the following
line to `/etc/sysctl.conf` (or uncomment it, if you have the default
Ubuntu `sysctl.conf`):

{% highlight text %}
net.ipv4.ip_forward=1 in /etc/sysctl.conf
{% endhighlight %}

Activate this sysctl setting with `sudo sysctl -p`. Next, we'll
configure iptables:

{% highlight text linenos %}
sudo iptables -A FORWARD -i eth0 -o eth1 -m state --state ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
{% endhighlight %}

As `root`, save this iptables ruleset:
`iptables-save >/etc/network/pxenet-nat.ipt`. Add a line to
`/etc/network/interfaces` under the `eth1` section to restore the
iptables rules when the interface comes up.

{% highlight text %}
        up /sbin/iptables-restore </etc/network/pxenet-nat.ipt
{% endhighlight %}

At this point our FAI server should be ready too bootstrap clients
with Debian. You can go ahead and create a new virtual machine and
give it a try if you'd like (or destroy and recreate the disk for the
"pxetext" VM and try it there). As I said, though, I'd like to be able
to bootstrap Ubuntu Server VMs. Although official support for "FAI
multi-distribution" isn't available in FAI 3.x, it works fine; we'll
just have to pull in a couple pieces of FAI 4.x.

The first thing we'll need is a "basefile" for the version of Ubuntu
we'd like to install. The config examples for FAI 4.x (available
[here][5]) include a `mk-basefile` shell script (under `basefiles/`)
which can be used to build the basefiles for your distribution of
choice. Currently, it supports Debian 6.0 (which we can install
out-of-the-box with FAI 3.x), Ubuntu 10.04, CentOS 5/6, and Scientific
Linux Cern 5/6. I want to install Ubuntu 10.10, so I need to patch the
shell script to add support.

{% highlight diff linenos %}
--- mk-basefile.orig      2011-09-07 17:27:24.000000000 -0700
+++ mk-basefile           2011-11-09 11:55:25.000000000 -0800
@@ -30,3 +30,3 @@
 MIRROR_DEBIAN=http://kueppers/debian/
-MIRROR_UBUNTU=http://ftp.halifax.rwth-aachen.de/ubuntu/
+MIRROR_UBUNTU=http://mirrors.kernel.org/ubuntu/
 MIRROR_CENTOS=http://mirror.netcologne.de/
@@ -37,3 +37,4 @@
 EXCLUDE_LUCID=dhcp3-client,dhcp3-common
-
+EXCLUDE_MAVERICK=dhcp3-client,dhcp3-common
+INCLUDE_MAVERICK=aptitude,tasksel

@@ -180,2 +181,12 @@

+maverick() {
+
+    local arch=$1
+
+    check
+    debootstrap --arch $arch --exclude=${EXCLUDE_MAVERICK} --include=${INCLUDE_MAVERICK} maverick $xtmp ${MIRROR_UBUNTU}
+    cleanup-deb
+    tarit
+}
+

@@ -191,2 +202,3 @@
     LUCID32      LUCID64
+    MAVERICK32   MAVERICK64
     SQUEEZE32    SQUEEZE64
@@ -224,2 +236,4 @@
     LUCID64) lucid amd64 ;;
+    MAVERICK32) maverick i386 ;;
+    MAVERICK64) maverick amd64 ;;
     SQUEEZE32) squeeze i386 ;;
{% endhighlight %}

<small> *You can download the patched file [here][6]* </small>

Create the ubuntu "basefile" (I want the 64-bit one):

{% highlight text %}
sudo ./mk-basefile -J MAVERICK64
{% endhighlight %}

Add a section to `/srv/fai/config/class/50-host-classes` for our test
Ubuntu installation:

{% highlight diff %}
--- 50-host-classes.orig      2011-11-09 14:16:09.000000000 -0800
+++ 50-host-classes           2011-11-09 14:14:15.000000000 -0800
@@ -31,2 +31,6 @@
        exit 0 ;; # CentOS/SLC does not use the GRUB class
+    ubuntu)
+        echo "FAIBASE UBUNTU"
+        ifclass I386 && echo "MAVERICK MAVERICK32"
+        ifclass AMD64 && echo "MAVERICK MAVERICK64" ;;
     *)
{% endhighlight %}

<small> *You can download the patched file [here][7]* </small>

Use the hostname you assigned earlier in `/etc/hosts`.

Next, we need to get the requisite apt files for the Ubuntu version we
want to install. You are probably safest getting them from a live CD
for the version of Ubuntu you want to install. You need `secring.gpg`,
`sources.list`, `trustdb.gpg`, and `trusted.gpg`. You may want to
modify `sources.list` to point at your favorite mirror (or a local
mirror if you'll be installing a lot of VMs). Here's a snippet that
will put these in the appropriate places in your FAI config:

{% highlight bash %}
for file in sources.list secring.gpg trustdb.gpg trusted.gpg; do sudo mkdir -p /srv/fai/config/files/etc/apt/${file}; sudo cp $file /srv/fai/config/files/etc/apt/${file}/MAVERICK; done
{% endhighlight %}

You'll also need to create
`/srv/fai/config/files/etc/network/interfaces/UBUNTU`:

{% highlight text %}
# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto eth0
iface eth0 inet dhcp
{% endhighlight %}

We also have to configure the packages we'll need to install. In
`/srv/fai/config/package_config/UBUNTU`:

{% highlight text %}
PACKAGES aptitude
apt
apt-utils
console-setup
debconf-utils
dhcp3-client
fai-client
grub
less
linux-image-generic
locales
openssh-server
vi

PACKAGES aptitude GRUB
grub lilo-

PACKAGES aptitude GRUB_PC
grub-pc grub- lilo-

PACKAGES aptitude LILO
lilo grub-
{% endhighlight %}

Just the basics, for now, we can add more packages later if we need
them. The most important one here is the "linux-image-generic"
package. We'll also want to rename
`/srv/fai/config/package_config/DEFAULT` to
`/srv/fai/config/package_config/DEBIAN`. The "DEFAULT" class is
applied to every FAI client, and since we're going to be working with
multiple distributions, we don't want to use the bundled DEFAULT
`package_config` file, which assumes a Debian client.

Finally, we need to create a [hook][8] for the `updatebase` step of
the installation to copy the apt files into place inside of the target
OS. The file should be `/srv/fai/config/hooks/updatebase.UBUNTU`:

{% highlight bash %}
#! /bin/bash
echo "Preparing apt for Ubuntu"
fcopy -v /etc/apt/secring.gpg
fcopy -v /etc/apt/sources.list
fcopy -v /etc/apt/trustdb.gpg
fcopy -v /etc/apt/trusted.gpg
{% endhighlight %}

`fcopy` is an FAI built-in that takes care of copying files from the
configuration space to the target OS for us.

At this point, you should be able to create a new VM similar to the
"pxetest" VM we created earlier, boot it, and watch FAI automagically
install Ubuntu onto the new VM. When the install completes, a message
is displayed telling you to hit Return to reboot. Before you do so you
may want to run `fai-chboot -d 192.168.0.10` (substituting the IP you
used for the new VM) on the "faiserver", and change the boot device
priority, so that the VM will boot from the disk instead of the
network. From here, you've got the basis of a pretty functional lab,
and you can start experimenting with your FAI configuration.

In Part III of this series, I'll show you how to use FAI to bootstrap
Redhat, and also set up our PXE server to allow us to boot a "live"
system for debugging. Redhat will be more of an adventure for me since
I'm pretty unfamiliar with the RPM-based distributions.

[1]: /2011/11/01/fai-lab-with-virtualbox-part-i.html
[2]: https://www.virtualbox.org/
[3]: http://fai-project.org/
[4]: http://fai-project.org/doc/man/fai-chboot.html
[5]: http://fai-project.org/download/misc/fai4-config.tar.gz
[6]: /files/failab/mk-basefile
[7]: /files/failab/50-host-classes
[8]: http://fai-project.org/fai-guide/ar01s07.html#_anchor_id_hooks_xreflabel_hooks_hooks
