# WireGuard - pfSense Tutorial

Wireguard - pfSense Tutorial to run a WireGuard server in your pfSense router installation. I will show you how to configure the actual server in pfSense, set up a remote host, and test our VPN connection.

This tutorial is mainly built for individuals that want a secure way to access their network and all its infrastructure, servers, & applications from outside their home network. I specifically chose this layout for my homelab because it allows me to emulate being inside of my network at home from the command line. I decided to go with WireGuard as my VPN protocol because of the speed and security compared to IPSec and OpenVPN.

In terms of ways to access your homelab resources outside your network, there are other options to chose from including [Cloudflare Tunnels](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps) and [Guacamole](https://guacamole.apache.org/doc/gug/introduction.html). I recommend that you check out each to decide which will be best for your own use case.


![MacOS Setup](https://github.com/SamG331/project-tutorials/blob/main/wireguard-pfsense/assets/WireGuard-Logo-Background.png?raw=true)


---

## Table of Contents
- [Requirements](#Requirements)
- [1. Server Configuration](#1. Server Configuration)
	- [[#1.  Server Configuration#1.1. Install WireGuard package on pfSense|1.1. Install WireGuard package on pfSense]]
	- [[#1.  Server Configuration#1.2. Configure WireGuard Firewall Rules|1.2. Configure WireGuard Firewall Rules]]
	- [[#1.  Server Configuration#1.3. Make WireGuard Tunnel|1.3. Make WireGuard Tunnel]]
	- [[#1.  Server Configuration#1.4. Configure WAN Firewall Rules|1.4. Configure WAN Firewall Rules]]
	- [[#1.  Server Configuration#1.5. Configure Outbound NAT|1.5. Configure Outbound NAT]]
- [2. Peer Configuration](#pc)
- [3. Testing & Troubleshooting|3. Testing & Troubleshooting](#tt)
- [[#Additional Notes|Additional Notes]]



---

## Requirements

- pfSense CE 2.5.2 & later OR pfSense Plus 21.05 & later

---

## 1. Server Configuration

First we need to install and configure Wireguard on out pfSense router/firewall applicance. This process will set up encryption keys for our server running on pfSense as well as configure rules to allow WireGuard traffic from set IPs and ports.

### 1.1. Install WireGuard package on pfSense

In the pfSense webConfigurator, go to **System > Package Manager > Available Packages** and install the WireGuard package 

### 1.2. Configure WireGuard Firewall Rules

We now want to configure rules for *any* WireGuard interface on our pfSense firewall. In this case we specifically want to allow traffic from our WireGuard peers that we will set up later on. 

>Under **Firewall > Rules > WireGuard** 

Add a new rule at the top:
- set **Action** to *Pass*
- set **Interface** to *WireGuard*
- set **Address Family** to *IPv4*
- set **Protocol** to *Any*
- **Description** - *Pass VPN traffic from WireGuard peers*

*If we miss this step you will find that our firewall will default to blocking all traffic on our Wireguard interface because of the way pfSense is set up to block all traffic when no firewall rules are in place

### 1.3. Make WireGuard Tunnel

Now we want to actually create to tunnel that will encrypt our traffic using WireGuard. This can be done in the webConfigurator as well and the setting we want are under **VPN > WireGuard**

>Under **Settings** tab
- check *Enable WireGuard*
- uncheck *Hide Peers*

>Under **Tunnels** tab

Add Tunnel:
- set **Description** to *WireGuard*
- set **Listen Port** to a custom port ( example: `51480` )
- click **Generate** to generate public and private key pairs
- set **Interface Addresses** to a custom private IPv4 designation (example: `172.16.16.1 /24`)

### 1.4. Configure WAN Firewall Rules

Now, importantly, we need to pass traffic through our new custom port to our WAN interface. For the sake of this tutorial I will continue using `51480` thoughout.

>Under **Firewall > Rules > WAN**

Add a new rule at the bottom:
- set **Action** to *Pass*
- set **Protocol** to *UDP*
- set **Destination** to *WAN Addess*
- set **Destination Port Range** to `51480 to 51480`
**Description** - *Allow WireGuard*

### 1.5. Configure Outbound NAT

Now there are other ways to handle NAT with WireGuard which include creating a dedicated interface for your specific WireGuard Tunnel. In this case we will configure an Outbound Nat rule in order to allow WireGuard to go out the WAN

>Under **Firewall > NAT > Outbound**

Select Outbound NAT Mode: 
- *Hybrid Outbound Nat rule generation*

Add Mapping:
- set **Address Family** to *IPv4*
- set **Source** to *Type: Network* & `172.16.16.0 /24`
**Description** - *Allow WireGuard to go out the WAN*

---

## 2. Peer Configuration <a name="pc"/>

Finally, we can now start adding and configuring "Peers" to our WireGuard server. In this example I am going to use MacOS client as my peer, but the process is nearly identical regardless or the device or OS.

1) On your Mac, go to the App Store and download WireGuard
2) ⌘ + n , and copy the public key generated from the new tunnel (do not close window yet)
3) Navigate to pfSense webConfigurator

>Under **VPN > WireGuard > Peers**

Add Peer:
- set **Tunnel** to *WireGuard*
- set **Public Key** to *public key from step 2*
- set **Allowed IPs** to `172.16.16.2 /32` 
	( IP we assign to our client, use /32 because each peer needs its own "space")
- **Description** - *MacOS*

4) Navigate back to WireGuard App on Mac and paste template

```bash
[Interface] #Host
	PrivateKey = <CLIENT_PRIVATE_KEY_ALREADY_IN_THE_FILE>
	Listen Port = 51480
	Address = 172.16.16.2/24 #the vpn address of the host */24 is intended
	DNS = 127.0.0.1, x.x.x.1 #the DNS of the peer & DNS of home network

[Peer] #HomeNetwork
	#taken from VPN => WireGuard => Tunnels (copy Public ID)
	PublicKey = <SERVER_PUBLIC_KEY>
	#find on Dashboard => Interfaces => WAN (port 51480)
	Endpoint = <your.public.ipv4.addr>:51480
	#the network of the vpn, and local networks
	AllowedIPs = 172.16.16.0/24, <x.x.1.0/24>, <x.x.10.0/24>, <x.x.20.0/24>
```

---

## 3. Testing & Troubleshooting <a name="tt"></a>

WireGuard is pretty quiet as far as VPN protocols go, there won't be alarms going off or many visible system errors if things are not configured properly. The way that we have this config set up, as a split tunnel, we want to make sure DNS is working from out remote network and our internal network.

#### Tests:

1) Connect to pfSense webConfigurator
`http:pfsense.yourlocalnetwork:`

4) Ping your HomeNetwork DNS server
```bash
ping x.x.1.1
```

3) Test host overrides (if your have them set up)
```bash
ssh root@web-server.yourlocalnetwork
```
- note that you must specify the FQDN of your home domain because of split tunnel mode

---

## Additional Notes

Since this tutorial was setting up a split tunnel config, if you would like a more "locked down" version where all traffic is routed through your HomeNetwork, you should go with a full tunnel config. PfSense has a [WIreGuard Recipe](https://docs.netgate.com/pfsense/en/latest/recipes/wireguard-client.html#) that will walk you through this process.

Lastly, for my purposes I did not create a dedicated interface for my tunnel in pfSense, because I simply do not need it but some benefits are...
-   Adds a firewall tab under **Firewall > Rules**
-   Allows the interface to be selected for use with NAT rules
-   Allows the interface to be selected throughout the GUI and packages for various purposes
-   Rules on assigned interface tabs get `reply-to` which ensures return routing will exit back the expected interface for inbound connections.