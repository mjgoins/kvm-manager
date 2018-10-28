KVM-Manager
===========

*Authors:*
 * Daniel Kahn Gillmor <dkg@fifthhorseman.net>
 * Jamie McClelland <jm@mayfirst.org>
 * Greg Lyle <greg@stealthisemail.com>

*Copyright:* © 2009-2011

*License:* GPL-3+

This is a small set of scripts to make it relatively easy to manage a
stable of kvm instances in a fairly secure and isolated fashion.

The basic model is to use runit or systemd to supervise each KVM instance,
with a single, non-privileged user account for each instance. You can login
via ssh as the non-privileged user and, via screen, access the instance's
console.

Dependencies
------------

 * `runit` or `systemd` : for system supervision
 * `kvm` : for the virtual machine emulator
 * `socat` : For communications with the monitor and console of guests
 * `screen` : for the detached, logged serial console
 * `bridge-utils` : for configuring a bridge device
 * `lvm2` : for creating the relevant block devices
 * `udev` : for configuring the block devices with proper permissions
 * `fakeroot` : for rebuilding the initramfs as a regular user in di-maker
 * `xorriso` : for grub2 to make an iso in di-maker
 * `genisoimage` : for di-maker to work with an existing iso
 * `sgabios` : for early pre-bootloader (like ipxe) output

Recommendations
+++++++++++++++

 * `openssh-server` : i've been using ssh to access the vm's serial console
 * `debian-installer-netboot` : having one of the d-i netboot packages installed makes guest installation easier
 
INSTALLATION
------------

 * Install dependencies:

        apt-get install kvm screen bridge-utils lvm2 udev socat sgabios

    If you want to use runit instead of systemd:

        apt-get install runit

    If you want to be able to use di-maker, you'll also need:

        apt-get install fakeroot xorriso grub2

 * Link programs into /usr/local/sbin:
 
        ln -s $(pwd)/{di-maker,kvm-manager,kvm-creator,kvm-start,kvm-setup,kvm-stop,kvm-teardown,kvm-screen} /usr/local/sbin/

 * Link screen configuration file into /etc

        ln -s $(pwd)/screenrc.kvm-manager /etc/

 * If using systemd, copy systemd service files into /etc/systemd/system

        cp $(pwd)/{kvm, kvm-screen}@.service /etc/systemd/system/

 * Configure your host network to use a bridge. If your network adaptor 
   is eth0, you can use the following in /etc/network/interfaces

        auto br0
        iface br0 inet static
          [Put your normal IP config for eth0 here...]
          hwaddress ether xx:yy:zz:aa:bb:cc
          bridge_ports eth0

    Note: explicitly setting the hwaddress of your bridge to the same
    MAC address as your existing NIC ("ip link show eth0 | grep
    ether") is a good idea -- it seems to avoid periods of network
    connectivity outages for the host when new interfaces get added to
    or removed from the bridge.

 * Alternately, you can create an internal-only bridge, and tell your
   host to pass traffic to it:

        auto br0
        iface br0 inet static
          [ internal IP address information ]
        post-up echo 1 > /proc/sys/net/ipv4/conf/br0/forwarding

INSTALLING DEBIAN ONTO YOUR VIRTUAL SERVER
------------------------------------------

To create a KVM instance, run:

    kvm-creator create $GUESTNAME [ $VG [$DISKSIZE [$RAM] ] ]

If you want to use systemd instead of runit:

    KM_PM=systemd kvm-creator create $GUESTNAME [ $VG [$DISKSIZE [$RAM] ] ]

You can replace "create" with "demo" to see the default values for non-
specified options.

The creator scripts creates a username and home directory, logical volume, and
the required directory in `/etc/sv/kvm/GUESTNAME` from which the kvm-manager
script is run. After creating your virtual server, you can modify the files in
`/etc/sv/kvm/GUESTNAME/env` to change initial settings.

You may also add ssh key's to `/home/GUESTNAME/.ssh/authorized_keys` to provide
additional access to other users.

At this point, your virtual server is created, however, it has no
operating system and it has not been started.

There are three options for installing debian onto the virtual server:

 * debian-installer
 * netboot
 * iso (like a CD install)


"debian-installer" is the simplest, but it requires that you have one
of the debian-installer-netboot packages installed
(e.g. `debian-installer-9-netboot-amd64`):

    touch /home/$GUESTNAME/vms/$GUESTNAME/debian-installer

To use the "netboot" method, make sure you have a working DHCP server
running on your host server and offering addresses over your bridge
interface.

Then, indicate that the server should boot via the network with:

    touch /home/$GUESTNAME/vms/$GUESTNAME/netboot

Finally, you can make a debian boot ISO image:

 *  Make the directory /usr/local/share/ISOs
 *  Create a serial console enabled debian installer.
   * cd /usr/local/share/ISOs
   * di-maker d-i.iso

Indicate that the server should boot via the CDROM (the equivelant of putting
the installer CD in the drive) with:

    ln -s /usr/local/share/ISOs/d-i.iso /home/$GUESTNAME/vms/$GUESTNAME/cd.iso

STARTING YOUR VIRTUAL SERVER
----------------------------

If you are using runit:

    update-service --add /etc/sv/kvm/$GUESTNAME

This process adds your virtual server to the runit service directory.

If you are using systemd:

    systemctl enable kvm@$GUESTNAME.service
    systemctl start kvm@$GUESTNAME.service

If `/home/$GUESTNAME/vms/$GUESTNAME/cd.iso` exists, the server will
behave as if you set the CDROM as the boot device in the bios.

If `/home/$GUESTNAME/vms/$GUESTNAME/netboot` exists, the server will
behave as if you set the network device as the boot device in the
bios.

After you have installed your server, be sure to delete these files if
they exist or your server won't boot properly.

ACCESSING YOUR VIRTUAL SERVER
-----------------------------

To access the guest's serial console, do:

    ssh -t $GUESTNAME@host.machine screen -x $GUESTNAME

To access the guest's KVM monitor, do:

    ssh -t $GUESTNAME@host.machine socat vms/$GUESTNAME/monitor.socket STDIO

HACKING
-------

All patches, fixes, suggestions welcome!
