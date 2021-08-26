# router-box

This box shows how to turn a Virtualbox Linux VM into a simple router. It relies
on Linux kernel ip forwarding feature. You can use this router as a network
proxy, and when combining with tools like `tcpdump`, you can monitor the network
traffic of other devices.

## Prerequisites

You must have below installed:

* Virtualbox
* Vagrant

Actually, Vagrant is optional, but at least you need to have a Linux VM. If you
don't have Vagrant, you can reference `Vagrantfile` to configure your VM.

> Note:
> The [Vagrantfile](Vagrantfile) is based on [my custom
> box](https://github.com/uzxmx/boxes/blob/master/mybox/Vagrantfile) which is based on
> Centos 7 box, so you can change the base box to Centos 7 and install
> software dependencies when needed.

## Quick start

**1. Boot up the router VM.**

Change to the root directory, and run `vagrant up`. For the fist time, it needs
to select a network interface to bridge to. If there are more than one
interfaces, it will ask you to select one.

When the router VM is running, execute `vagrant ssh`.

**2. Check if the router works by a test VM.**

Change to the `test` directory, and run `vagrant up`. It will bridge to the same
network interface as the router VM and update its gateway to the router IP. When
the test VM is running, execute `vagrant ssh`.

In the router VM, run `sudo tcpdump -i eth1 src 8.8.8.8 or dst 8.8.8.8`.

In the test VM, run `ping 8.8.8.8`.

Switch to the router VM, when you see something like below, then the router
works.

> In the following, `192.168.3.11` is the IP of test VM in my laptop, and
> `192.168.3.17` is the IP of my laptop which the two VMs bridges to.

```
17:59:03.278912 IP 192.168.3.11 > dns.google: ICMP echo request, id 4108, seq 3, length 64
17:59:03.282524 IP 192.168.3.17 > dns.google: ICMP echo request, id 4108, seq 3, length 64
17:59:03.338418 IP dns.google > 192.168.3.17: ICMP echo reply, id 4108, seq 3, length 64
17:59:03.338515 IP dns.google > 192.168.3.11: ICMP echo reply, id 4108, seq 3, length 64
```

**3. Check if the router works by a mobile device.**

Connect the mobile device to the same LAN as the router VM. When connected,
manually change the gateway (router) address to the IP of the router VM.

You also need to manually change the DNS server to a proper one, such as the IP
of the real router in your LAN (not the VM router), or google dns service. This
is because client can only get DNS servers in a DHCP message when DHCP is used,
not when a static IP address is manually assigned.

In the router VM, run below with `YOUR-MOBILE-DEVICE-IP` updated:

```
sudo tcpdump -i eth1 src <YOUR-MOBILE-DEVICE-IP> or dst <YOUR-MOBILE-DEVICE-IP>
```

Use your device to visit some websites. If you see some packets dumped, then
your router VM is working.

```
# For mac osx, get current gateway address:
route -n get default

# For linux,
ip route show dev eth0
```

## License

[MIT License](LICENSE)
