---
layout: post
title: FAI Lab With VirtualBox (Part I)
---

The first project I've been asked to work on here at [Yammer][4] is
improving our provisioning process. Right now there are several places
where too much manual intervention is involved in getting new servers
provisioned. This isn't the first time I've been down this block, and
sure enough, my tool of choice is the one already in place:
[FAI][1]. My task is to clean up our FAI install, streamline things a
bit, and give us the flexibility to bootstrap several Linux
distributions.

The problem with developing an FAI install is that it is difficult to
do on a live network without either causing disruptions (multiple DHCP
servers on the same network == sad panda) or having to go through a
lot of trouble to set up an isolated network segment. One of my
coworkers mentioned that he has, in the past, set up a virtual network
using [VirtualBox][2] to use as a lab for development of FAI
configurations. I decided I'd give this a try. There's a lot of
documentation out there about all of the components, but I didn't find
anything that drew them all together, so I thought I'd do that here.

### Setting up the FAI server VM

The first thing we need to do is set up a [host-only network][5] in
VirtualBox. This creates the virtual network that our FAI server and
clients will use to talk to each other. I'm not sure why VirtualBox
refers to this as "host-only networking" - I find the terminology
confusing.

{% highlight bash linenos %}
# Note the interface name output by this command, it will be vboxnetN,
# where 'N' is a number.
VBoxManage hostonlyif create
# Using 'N' from the previous command, run: `VBoxManage hostonlyif
# ipconfig vboxnetN --ip 192.168.N.1` for example, if N = 0
VBoxManage hostonlyif ipconfig vboxnet0 --ip 192.168.0.1
{% endhighlight %}

Line 3 creates a new host-only network interface in the VirtualBox
subsystem, while line 6 sets the IP for the host-only network
interface just created. It's important to remember that this IP
address is the IP of the *host*, not the virtual machine. The host
machine is connected to the "host-only network" as well as the other
virtual machines which are configured to be part of that network.

The next step is configuring our FAI server. We'll assume that the
host-only network you created in the previous step is `vboxnet0`.

{% highlight bash linenos %}
VBoxManage createvm --name faiserver --ostype Ubuntu_64 --register
VBoxManage modifyvm faiserver --memory 512 --boot1 disk --boot2 dvd --boot3 net --rtcuseutc on
VBoxManage modifyvm faiserver --nic1 nat --nic2 hostonly --hostonlyadapter2 vboxnet0
VBoxManage createhd --size 10000 --filename $HOME/Library/VirtualBox/HardDisks/faiserver-root
VBoxManage storagectl faiserver --name ide0 --add ide --controller PIIX4
VBoxManage storageattach faiserver --storagectl ide0 --device 0 --port 0 --type hdd --medium $HOME/Library/VirtualBox/HardDisks/faiserver-root.vdi
{% endhighlight %}

Let me break this down line-by-line:

   1. This command creates a new VM named "faiserver" in the
      VirtualBox subsystem. I intend to install a 64-bit Ubuntu on
      this VM, and chose the ostype argument accordingly. To learn
      about the various identifiers that can be used here, use:
      `VBoxManage list ostypes`
   1. Relatively self-explanatory; we give the VM 512M of RAM, set the
      boot order, and tell it to use UTC for the hardware clock.
   1. Continuation of the previous line.
   1. Here we create virtual network interfaces for our VM. The first
      NIC will use VirtualBox [NAT][6] networking, allowing it to
      communicate with the outside world via the host. The second
      interface will be connected to the host-only network we created
      earlier.
   1. Create a 10G virtual hard disk for our VM.
   1. Create a virtual hard disk controller for our VM.
   1. Attach the virtual hard disk to our virtual controller.

Install your OS of choice. I'm using [Ubuntu Server 10.10][3], and I
installed it by downloading an ISO and attaching it to the VM as a
virtual DVD drive, booting from the virtual DVD drive, and doing a
base installation:

{% highlight bash linenos %}
# Attach the ISO image as a virtual DVD drive.
VBoxManage storageattach faiserver --storagectl ide0 --device 1 --port 0 --type dvddrive --medium $HOME/Downloads/ubuntu-10.10-server-i386.iso
# Before proceeding, boot the VM and install the OS. Then shut down the VM.
# Replace the ISO with a virtual empty DVD drive.
VBoxManage storageattach faiserver --storagectl ide0 --device 1 --port 0 --type dvddrive --medium emptydrive`
{% endhighlight %}

The last command isn't really necessary; if you skip this step
VirtualBox will always attach the ISO when you boot the VM, which I
find annoying.

### Setting up port forwarding for SSH

If you don't mind working within the VirtualBox window, you can skip
to "Configuring the Virtual Network". However, I find it easiest to
run VirtualBox VMs headless and access them over ssh, so I set up port
forwarding on the NAT interface and a stanza in my `~/.ssh/config` to
make it easy to connect to the FAI server.

{% highlight bash %}
VBoxManage modifyvm faiserver --natpf1 ",tcp,,2200,,22"
{% endhighlight %}

This command forwards localhost:2200 to port 22 on the "faiserver"
VM. For more details on the syntax here, you'll want to look at
"[Configuring port forwarding with NAT][7]" in the VirtualBox User
Manual. *Note: you can't set up port forwarding while the VM is
running for some reason, so do this before you start up the VM again.*

And here's the stanza for your `~/.ssh/config`:

    Host faiserver
        HostName localhost
        Port 2200

This lets you connect to the FAI server with `ssh faiserver` instead
of having to remember the port you forwarded. Now that we've finished
setting up remote access, you can start your VM in headless mode and
connect over ssh:

{% highlight bash %}
VBoxManage startvm faiserver --type headless
ssh faiserver
{% endhighlight %}

### Configuring the Virtual Network

Now that you have your FAI server up and running, we'll want to get it
connected to the host-only network we configured at the very
beginning. I'll assume your virtual network is `vboxnet0` and that you
configured the IP address as `192.168.0.1` on the host. To get your
"faiserver" VM connected to that network, run this command on the VM:

{% highlight bash %}
ip addr add 192.168.0.2/16 dev eth1
{% endhighlight %}

This assigns the "faiserver" VM the IP `192.168.0.2` on the virtual
network. See if you can communicate with the host:

{% highlight text %}
faiserver ~ » ping -c3 192.168.0.1
PING 192.168.0.1 (192.168.0.1) 56(84) bytes of data.
64 bytes from 192.168.0.1: icmp_req=1 ttl=63 time=0.207 ms
64 bytes from 192.168.0.1: icmp_req=2 ttl=63 time=0.275 ms
64 bytes from 192.168.0.1: icmp_req=3 ttl=63 time=0.297 ms

--- 192.168.0.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 1998ms
rtt min/avg/max/mdev = 0.207/0.259/0.297/0.042 ms
{% endhighlight %}

If you don't get a response to ping, something has gone wrong;
double-check to be sure you've run all the network-related commands as
specified. To make this configuration permanent (on Ubuntu, at least),
edit `/etc/network/interfaces`. The result should look something like
this:

{% highlight text linenos %}
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
# This is connected to VirtualBox via NAT
auto eth0
iface eth0 inet dhcp

# The secondary network interface
# This is connected to the host-only network vboxnet0
auto eth1
iface eth1 inet static
        address 192.168.0.2
        netmask 255.255.0.0
{% endhighlight %}

This should make sure the virtual network is configured whenever the
"faiserver" VM is booted.

### Installing FAI

Installation of FAI itself is easy; there are packages available for
Ubuntu which are a new enough version to be usable.

{% highlight bash %}
sudo aptitude install fai-server fai-doc
{% endhighlight %}

FAI's configuration is controlled by files located in `/etc/fai`. The
[FAI manual][8] does a great job of outlining these files and their
purpose; I'll only touch on the ones we're modifying. First, we'll
take a look at `/etc/fai/make-fai-nfsroot.conf`:

{% highlight diff %}
--- make-fai-nfsroot.conf.orig       2010-10-05 18:04:47.000000000 -0700
+++ make-fai-nfsroot.conf            2011-11-03 19:36:38.587117092 -0700
@@ -14,5 +14,5 @@
 # Add a line for mirrorhost and installserver when DNS is not available
 # on the clients. This line(s) will be added to $nfsroot/etc/hosts.
-#NFSROOT_ETC_HOSTS="192.168.1.250 yourinstallserver"
+NFSROOT_ETC_HOSTS="192.168.0.2 faiserver"

 # Parameter for debootstrap: "<suite> <mirror>"
{% endhighlight %}

The only modification we need to make is to add the IP address of our
"faiserver" VM to `NFSROOT_ETC_HOSTS` so that our netbooted clients
will know where to find it. You should use the IP address you assigned
in the previous section.

You may be asking yourself "What about the `FAI_DEBOOTSTRAP` variable?
Aren't we using Ubuntu?" This is one of the points where it is easy to
get yourself turned around with FAI. There are three separate
operating systems instances you are dealing with. There is the OS on
the FAI server, which I will call the "FAI server OS." There is the OS
which you will boot a system into in order for FAI to run. This is the
OS which is wrapped up in the FAI nfsroot. I'll call this the "FAI
client OS." Finally, there is the OS you want installed on the system
**after** FAI has completed. I'll call this the "target OS." The
reason we don't want to mess with `FAI_DEBOOTSTRAP` is because FAI is
designed to run under Debian; it is just simpler to use Debian for
your FAI client OS." I got myself confused here by trying to install
Ubuntu in the nfsroot and wasted several hours.

Now that we've dealt with that digression, we can look at
`/etc/fai/NFSROOT`. This file specifies the packages which should be
installed within the FAI client OS. As shipped, FAI specifies
linux-image as the package for the kernel. Since linux-image is a
virtual package, this ends up with no kernel and no initrd installed
in your nfsroot. Here's what I needed to change to make things work
correctly:

{% highlight diff linenos %}
--- NFSROOT.orig   2010-10-05 18:04:47.000000000 -0700
+++ NFSROOT        2011-11-03 19:49:26.755111976 -0700
@@ -18,3 +18,3 @@
 grub lilo read-edid
-linux-image
+linux-image-686

@@ -22,3 +22,3 @@
 grub lilo
-linux-image
+linux-image-amd64

{% endhighlight %}

We can do a quick check to make sure everything is working at this
point. Run `sudo make-fai-nfsroot`:

{% highlight text linenos %}
Creating FAI nfsroot in /srv/fai/nfsroot/live/filesystem.dir.
By default it needs more than 380 MBytes disk space.
This may take a long time.
/srv/fai/nfsroot/live/filesystem.dir already exists. Removing /srv/fai/nfsroot/live/filesystem.dir
Creating base system using debootstrap version 1.0.23ubuntu1
Calling debootstrap squeeze /srv/fai/nfsroot/live/filesystem.dir http://cdn.debian.net/debian
Creating base.tgz
Upgrading /srv/fai/nfsroot/live/filesystem.dir
install_packages: reading config files from directory /etc/fai
Adding additional packages to /srv/fai/nfsroot/live/filesystem.dir:
nfs-common fai-nfsroot module-init-tools ssh rdate lshw portmap rsync lftp less dump reiserfsprogs e2fsprogs usbutils psmisc pciutils hdparm smartmontools parted mdadm lvm2 dnsutils ntpdate dosfstools jove xfsprogs xfsdump procinfo dialog discover console-setup iproute udev subversion liblinux-lvm-perl cfengine2 libapt-pkg-perl grub lilo read-edid linux-image-686
install_packages: reading config files from directory /etc/fai
Extracting templates from packages: 100%
install_packages exit code: 0
`/etc/fai/apt' -> `/srv/fai/nfsroot/live/filesystem.dir/etc/fai/apt'
`/etc/fai/apt/sources.list' -> `/srv/fai/nfsroot/live/filesystem.dir/etc/fai/apt/sources.list'
`/etc/fai/fai.conf' -> `/srv/fai/nfsroot/live/filesystem.dir/etc/fai/fai.conf'
`/etc/fai/grub.cfg' -> `/srv/fai/nfsroot/live/filesystem.dir/etc/fai/grub.cfg'
`/etc/fai/live.conf' -> `/srv/fai/nfsroot/live/filesystem.dir/etc/fai/live.conf'
`/etc/fai/make-fai-nfsroot.conf' -> `/srv/fai/nfsroot/live/filesystem.dir/etc/fai/make-fai-nfsroot.conf'
`/etc/fai/menu.lst' -> `/srv/fai/nfsroot/live/filesystem.dir/etc/fai/menu.lst'
`/etc/fai/NFSROOT' -> `/srv/fai/nfsroot/live/filesystem.dir/etc/fai/NFSROOT'
Shadow passwords are now on.
`/srv/fai/nfsroot/live/filesystem.dir/boot/vmlinuz-2.6.32-5-686' -> `/srv/tftp/fai/vmlinuz-2.6.32-5-686'
`/srv/fai/nfsroot/live/filesystem.dir/boot/initrd.img-2.6.32-5-686' -> `/srv/tftp/fai/initrd.img-2.6.32-5-686'
DHCP environment prepared. If you want to use it, you have to enable the dhcpd and the tftp-hpa daemon.
Running hooks...
Removing 'local diversion of /sbin/discover-modprobe to /sbin/discover-modprobe.distrib'
make-fai-nfsroot finished properly.
No diversion 'any diversion of /sbin/discover-modprobe', none removed.
Log file written to /var/log/fai/make-fai-nfsroot.log
{% endhighlight %}

The ouput is pretty noisy, but at this point all we really want to see
is that the process completes without errors. Pay particular attention
to lines 24 and 25; this is where the kernel and initrd are copied, if
you see any errors here you will need to fix them before moving
on. Otherwise, we're ready to finalize the FAI setup and export our
nfsroot. Run `fai-setup` to generate the final nfsroot, add it to
`/etc/exports`, and restart NFS. This command is just as noisy as
`make-fai-nfsroot` was. Watch for "make-fai-nfsroot finished
properly." and "FAI setup finished." in the output; if you don't see
these two messages, you probably have some errors to deal with.

`fai-setup` will unfortunately assume that the first network interface
is the one we're exposing NFS from, so we'll need to edit the line in
`/etc/exports` to replace the IP address of the first interface with
the IP address we assigned to eth1 for connecting to the virtual
network. Your `/etc/exports` line should end up looking like this:

{% highlight text %}
# /etc/exports: the access control list for filesystems which may be exported
#               to NFS clients.  See exports(5).
#
# Example for NFSv2 and NFSv3:
# /srv/homes       hostname1(rw,sync,no_subtree_check) hostname2(ro,sync,no_subtree_check)
#
# Example for NFSv4:
# /srv/nfs4        gss/krb5i(rw,sync,fsid=0,crossmnt,no_subtree_check)
# /srv/nfs4/homes  gss/krb5i(rw,sync,no_subtree_check)
#
/srv/fai/nfsroot 192.168.0.2/16(async,ro,no_subtree_check,no_root_squash)
{% endhighlight %}

After editing `/etc/exports`, run `sudo exportfs -a` to export the
nfsroot properly.

### Configuring the DHCP server for PXE booting

In order to PXE boot our FAI clients we'll need to run a DHCP server
as well as a TFTP server. The `fai-server` package installs
`dhcp3-server` and `tftpd-hpa`; to match the environment I'm
developing for, I'm going to be using [dnsmasq][9] instead. I don't
necessarily advocate the use of dnsmasq, you should evaluate the
choices and pick what works for your environment. That said, I'll
detail my setup here so you have a complete example to work with.

First, let's get rid of those packages that were installed for us:
`sudo aptitude purge dhcp3-server tftpd-hpa`. Then install dnsmasq:
`sudo aptitude install dnsmasq`. Create `/etc/dnsmasq.d/pxenet` with
the following contents.

{% highlight text linenos %}
port=0
interface=eth1
dhcp-range=192.168.0.10,static,7d
read-ethers
dhcp-ignore=tag:!known
dhcp-boot=pxelinux.0
log-dhcp
enable-tftp
tftp-root=/srv/tftp/fai
{% endhighlight %}

Broken down line-by-line:

   1. Disable listening on the DNS port. We're not going to be using
      dnsmasq for DNS requests.
   1. Only listen on the interface we have connected to the virtual
      network.
   1. Pass out IP addresses from our virtual network. `static` means
      that only hosts which have static addresses from `/etc/ethers`
      will be served. `7d` means the lease time will be 7 days.
   1. Read `/etc/ethers` for information about hosts for the DHCP
      server. This provides a static mapping of MAC address to IP.
   1. Ignore requests from hosts which are not known (i.e. listed in
      `/etc/ethers`). dnsmasq sets the "known" tag automatically.
   1. Filename for network booting.
   1. Enable extra logging for DHCP; allows us to debug any problems.
   1. Enable the TFTP server.
   1. Set the root directory for the TFTP server to the location FAI
      places the requisite files.

Restart dnsmasq (`/etc/init.d/dnsmasq restart`) and it will pick up
the new configuration. You may want to tail `/var/log/daemon.log` to
watch for any error messages from dnsmasq.

At this point we should have a working FAI server ready to PXE boot
our first client. In [Part II][10] of this series we'll make sure PXE
booting is working, and set up FAI to bootstrap and configure Ubuntu
Server.

[1]: http://fai-project.org/
[2]: https://www.virtualbox.org/
[3]: http://mirror.liberty.edu/pub/ubuntu-releases/10.10/
[4]: http://www.yammer.com
[5]: https://www.virtualbox.org/manual/ch06.html#network_hostonly
[6]: https://www.virtualbox.org/manual/ch06.html#network_nat
[7]: https://www.virtualbox.org/manual/ch06.html#natforward
[8]: http://fai-project.org/fai-guide/
[9]: http://thekelleys.org.uk/dnsmasq/doc.htm
[10]: /2011/11/08/fai-lab-with-virtualbox-part-ii.html
