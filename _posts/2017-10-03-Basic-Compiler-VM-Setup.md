---
layout: post
title:  "Basic Compiler VM Setup"
date:   2017-10-04 15:00:00 +0200
categories: "FreeBSD Development Setup"
---

In this blog post, I will go through the basic setup of the VM which I will then use
as compiler. It is *not* about setting up the PXE boot server, but network and basic setup.

# Prerequisites
* Installed QEMU with KVM support
* FreeBSD installer image

# Part 1: Network Setup
We will setup a virtual network for the Machines to communicate with
each other and the outside world. While it may seem tempting to just use QEMUs `-net user` as
a starting point, the real network setup will have to be done eventually too as `-net user` interferes
with PXE, so it is best to start with the real thing from the get go.

The setup will consist of a bridge between tap interfaces which the clients will use, and that will act as gateway to the host, as well as iptables rules to allow the machines to reach the outside network by being NATed by the host.

I am no network administrator, so please take the iptables rules with a grain of salt, they work
for me and I don't think they will introduce severe holes to your configuration. If they are wrong
in any way or you can recommend improvements, PRs are highly appreciated as always.

I packed the whole setup in a script, which is currently of rather low quality, but it does the job:

{% highlight shell %}
if [ "$EUID" = 0 ]; then
    echo "Do not run this as root, it fucks up the whoami otherwise!"
    exit
fi

outAdapter=wlp4s0

sudo ip link add name VMBridge type bridge
sudo ip tuntap add dev VMTap0 mode tap user $(whoami)
sudo ip tuntap add dev VMTap1 mode tap user $(whoami)
sudo ip link set VMTap0 master VMBridge
sudo ip link set VMTap1 master VMBridge
sudo ip link set VMBridge up
sudo ip link set VMTap0 up
sudo ip link set VMTap1 up
sudo ip addr add 192.168.0.100/24 dev VMBridge
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -t nat -A POSTROUTING -s 192.168.0.0/24 -o $outAdapter -j MASQUERADE
sudo iptables -A FORWARD -i VMBridge -s 192.168.0.0/24 -o $outAdapter -j ACCEPT
sudo iptables -A FORWARD -d 192.168.0.0/24 -m state --state ESTABLISHED,RELATED -i $outAdapter -j ACCEPT
{% endhighlight %}


# Part 2: Installing FreeBSD
Now, create a new virtual hard drive like so:

{% highlight shell %}
qemu-img create -f qcow2 clean.img 50G
{% endhighlight %}

and then launch a VM, using our brand new network setup.

{% highlight shell %}
qemu-system-x86_64 -net nic,model=virtio -net tap,ifname=VMTap0,script=no,downscript=no,vhost=on -m 4096 -enable-kvm -smp 2 -hda clean.img -cdrom <path to .iso>
{% endhighlight %}

Now work your way through the the installation, here are the important steps:
* Install all optional system components (we will need some later, but take all, just in case) 
* During Network Configuration, choose our `VirtIO Networking Adapter` and configure the following
    - Yes to IPv4
        - No to DHCP
        - Set IP address to: `192.168.0.1`
        - Subnet Mask to `255.255.255.0`
        - Default Router to `192.168.0.100` (The IP address of our bridge)
    - No to IPv6, since the network setup currently does not deal with that
    - Resolver Configuration
        - Search: `vmnet.local`, does not really matter for us
        - IPv4 DNS #1: `8.8.8.8` (or whatever is your favorite)
        - IPv4 DNS #2: `8.8.4.4`
* As services to start at boot choose sshd as well as dumpdev
* Adding users - you could, but what for at the moment?
* Yes to manual configuration
    - Edit `/etc/ssh/sshd_config` (for example using `ee`) to allow password based root login, by setting `PermitRootLogin yes` (in case you wondered, this editor allows saving on quit, so just choose leave, and you will be prompted to save)
    - Exit and reboot, but instead booting back into the vm, kill qemu when booting again. 

# Part 3: SSH with keys + finishing our setup

From now on, start the VM with the following command, which you can also pack into a shell script to launch the VM easier:

{% highlight shell %}
qemu-system-x86_64 -net nic,model=virtio -net tap,ifname=VMTap0,script=no,downscript=no,vhost=on -m 4096 -enable-kvm -smp 2 -hda clean.img
{% endhighlight %}

On your host computer, generate a new key pair for use with the VMs

{% highlight shell %}
ssh-keygen -f ~/.ssh/freeBSDVMsKey
{% endhighlight %}

and then copy it to the VM via

{% highlight shell %}
ssh-copy-id -i ~/.ssh/freeBSDVMsKey root@192.168.0.1
{% endhighlight %}

With the key on the machine, only allow root login via the key, by setting `PermitRootLogin without-password`.

Now is a good time to install your favourite shell and editor to the VM.

And that's it. We now have a clean FreeBSD installation as a starting point. Shut down and create a snapshot of it!

{% highlight shell %}
qemu-img create -f qcow2 -b clean.img pxeboot.img
{% endhighlight %}

**ATTENTION:** From now on, use the `pxeboot.img`, as using `clean.img` would lead to corruption since `pxeboot.img` is linked to it.


**Optional:** You can also create an entry in your `.ssh/config`-file to make ssh'ing to the machine more comfortable:

{% highlight shell %}
Host compiler
    HostName 192.168.0.1
    User root
    IdentityFile ~/.ssh/freebsdVMs
{% endhighlight %}

Now you only have to `ssh compiler` :)


