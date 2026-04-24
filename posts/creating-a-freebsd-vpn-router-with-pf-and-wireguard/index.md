<!--
.. title: Creating a FreeBSD VPN Router with PF and Wireguard
.. slug: creating-a-freebsd-vpn-router-with-pf-and-wireguard
.. date: 2026-04-22 21:47:19 UTC-07:00
.. tags: freebsd
.. category: 
.. link: 
.. description: 
.. type: text
-->

## Introduction

It is occasionally useful to forward all traffic from a particular subnet over a vpn for privacy.

## Assumptions

I am assuming that you already have FreeBSD installed, and a working Wireguard configuration that you might have got from a VPN provider, or perhaps crafted yourself. The idea is that all traffic behind the FreeBSD VPN Router will be "protected" by the vpn traffic. Therefore, we will run DHCP on this subnet.

<!-- TEASER_END -->

## Install packages

```bash
pkg install -y wireguard-tools dnsmasq
```

## Configure Wireguard

Create the file `/usr/local/etc/wireguard/wg0.conf` with your VPN configuration. Here is mine:
```ini
[Interface]
PrivateKey = redacted
Address = redacted

# Important so wireguard doesnt add routes automatically
Table = off

[Peer]
PublicKey = redacted
AllowedIPs = 0.0.0.0/0
Endpoint = redacted:51820
PersistentKeepalive = 25
```

## Configure PF

Create the following `/etc/pf.conf`...adapt to your use case:
```ini
ext_if="vtnet0"                                                      # Connection to internet
int_if="vtnet1"                                                      # VPN Protected interface
vpn_if="wg0"                                                         # VPN interface
local_subnets="172.16.30.0/24"                                       # VPN Protected interface subnet

scrub in

nat on $vpn_if from $int_if:network to !$int_if:network -> ($vpn_if) # Nat all traffic from $int_if:network to $vpn_if

block log all                                                        # Block the rest
pass on lo0                                                          # Skip the loopback
pass out keep state                                                  # Pass traffic out of this router
pass proto icmp                                                      # Pass ICMP
pass in on $int_if route-to $vpn_if from $int_if:network keep state  # Pass traffic to VPN
pass inet proto udp from any to any port 67:68                       # Allow DHCP
pass in on $ext_if proto tcp from $ext_if:network to self port ssh   # Allow SSH from $ext_if
```

## Configure Dnsmasq

I'm using Dnsmasq for dhcp only, so here is my `/usr/local/etc/dnsmasq.conf`:
```ini
# Disable dns functionality
port=0

# Only bind on vtnet1 (VPN Protected interface)
interface=vtnet1

# DHCP Range for your VPN Protected interface
dhcp-range=set:mag,172.16.30.100,172.16.30.200,255.255.255.0,2h

# Replace with your preferred dns server for the protected subnet!
dhcp-option=6,8.8.8.8 
```

## Configure rc.conf services and networking

Here is what my `/etc/rc.conf` looks like:
```bash
hostname="freebsdvpnrouter"

# Unprotected "WAN" interface. Could be just your normal network.
ifconfig_vtnet0="DHCP"

# Protected interface. Make up any subnet you want. This is mine.
ifconfig_vtnet1="inet 172.16.30.1/24"

# Services
dnsmasq_enable="YES"
gateway_enable="YES" # Required for routing functionality!!
pf_enable="YES"
sshd_enable="YES"
wireguard_enable="YES"
wireguard_interfaces="wg0"
```

## Bringing it all together

You should `reboot` to bring all services up after updating `/etc/rc.conf`. After that, you can connect a host (virtual or otherwise) to your protected vpn interface. It should get an ip on your new subnet. Make sure that the public ip seen on the VPN Protected Interface is different that your public one. My favorite way is curl:
```bash
curl http://ifconfig.me
```

This also validates DNS is working. You could also use a browser.


## Troubleshooting

### Verify the VPN is connected

Use the wg command to validate. Here is mine in the working state:
```bash
root@freebsdvpnrouter--> wg
interface: wg0
  public key: redacted
  private key: (hidden)
  listening port: 47619

peer: redacted
  endpoint: redacted:51820
  allowed ips: 0.0.0.0/0
  latest handshake: 2 minutes ago
  transfer: 188.32 MiB received, 6.61 MiB sent
  persistent keepalive: every 25 seconds
```
