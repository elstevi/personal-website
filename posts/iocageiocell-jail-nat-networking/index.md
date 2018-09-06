<!--
.. title: iocage/iocell jail nat networking
.. slug: iocageiocell-jail-nat-networking
.. date: 2018-03-27 23:06:31 UTC-07:00
.. tags: bridge, freebsd, jails, pf
.. category: 
.. link: 
.. description: 
.. type: text
-->

Giving jails their own subnet and routing traffic into that subnet with PF has some benefits. It allows jails to communicate freely between themselves, but keep that traffic on the backend private subnet. I like exposing an ssh service running in a jail on the backend, and using -D flag of ssh to proxy some ssh and http traffic to the backend network. Kind of a poor mans development environment and VPN.
<!-- TEASER_END -->
<h2>Create backend subnet</h2>

The subnet I will be using in this example is 172.16.17.0/24. The jail host will be using the gateway IP 172.16.17.1. I'll create the bridge for the next reboot:

```
sysrc cloned_interfaces+="bridge0"
sysrc ifconfig_bridge0="172.16.17.1/24"
```



And create it on the live system:

```
ifconfig bridge0 create
ifconfig bridge0 172.16.17.1/24 up
```


<h2>Configure traffic routing into and out of the backend subnet</h2>

Configure pf like this: (/etc/pf.conf)

```
# External interface, the wan side of the nat
ext_if = "cxgb0"

# Internal interface
int_if = "bridge0"
localnet = $int_if:network

ports_to_forward="{ 80, 443 }"
forward_host="172.16.17.10"

nat on $ext_if inet from $localnet to any -> ($ext_if)
rdr on $ext_if proto tcp from any to any port $ports_to_forward -> $forward_host

pass from { self, $localnet } to any keep state
pass in on $ext_if proto {udp, tcp} from any to any port $ports_to_forward keep state

set skip on bridge0
```

Enable PF on next boot and on the live system:
```
sysrc pf_enable="YES"
service pf start
```

Also, enable gateway on boot and on the live system:
```
sysrc gateway_enable="YES"
service gateway start
```
This enables the FreeBSD machine to forward ipv4 packets between subnets it sees.

<h2>Create jail and configure it to have an IP on the bridge</h2>
This will create a jail on the subnet with access
```
iocell create tag=testjail ip4_addr="bridge0|172.16.17.10" defaultrouter="172.16.17.1" boot=on
iocell start testjail
iocell console testjail
```

Next, to ensure that inbound traffic on ports 80 and 443 into to the WAN ip, I just installed nginx and started it up.
```
pkg install -y nginx
service nginx onestart
```

Done!
