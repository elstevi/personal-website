<!--
.. title: Configuring iocage for vnet and dhcp on FreeBSD 12 
.. slug: vnet-iocage-freebsd-12
.. date: 2019-03-21 22:22:26 UTC-07:00
.. tags: vnet vimage iocage freebsd 12.0 dhcp
.. category: 
.. link: 
.. description: 
.. type: text
-->

FreeBSD 12.0 shipped with vnet networking enabled by default. This allows a jail to be allocated a real fake interface, and for it to have its own networking stack. I have been using iocage, so here is how I configure a jail:

<!-- TEASER_END -->

## Create the jail

```
iocage create --name mgmt_jail --release 12.0-RELEASE
```
## Set misc networking settings
```
iocage set vnet=on mgmt_jail
iocage set dhcp=on mgmt_jail
iocage set bpf=yes mgmt_jail
```
## Map vnet interfaces to bridges
```
iocage set interfaces=vnet0:bridge1001 mgmt_jail
```

## Disable iocage default networking
This will optionally add your default interface to bridge0 and set up the jail to listen on that. Great for alot of use cases, but not this time.

```
iocage set vnet_default_interface=none mgmt_jail
```
## Start the jail!
```
iocage start mgmt_jail
```
```
root@r2--> iocage start mgmt_jail
* Starting mgmt_jail
  + Started OK
  + Using devfs_ruleset: 7
  + Configuring VNET OK
  + Starting services OK
  + DHCP Address: 10.1.1.117/24
```

## All of this at creation time

You can do all of this at creation time, its just less readable:
```
iocage create --name mgmt_jail --release 12.0-RELEASE vnet=on dhcp=on bpf=yes interfaces=vnet0:bridge1001 vnet_default_interface=none
```
