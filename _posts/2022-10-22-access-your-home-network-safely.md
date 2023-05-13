---
layout: post
title:  "Access your home network safely"
description: learn how to access your home network from anywhere safely
tags: security guide
---

As people are starting to work from everywhere, one of the most common questions that I get is always related to access files. No one wants to have duplicates of the same file or having to manually get your backup synced. Working from everywhere is great, but you have to make sure that you keep your property’s safe and closed to the bad internet people. There are multiple ways you can achieve that, but I think everyone agrees that using your own VPN connection back to your home network is probably the safest solution — and it’s not that hard to implement. For this use case, we’ll look at implementing a WireGuard server on your home router, as this is one of the most lightweight and secure solutions out there currently.

# Advantages
- Access everything you have on your home network (file shares, movies, photos, etc.) from anywhere in the world, as long as you have an internet connection
- Back up your files/devices even if you’re not at home
- Protect your online presence on the go

# Requirements
There are multiple ways you could go with implementing a WireGuard server, but we’ll try to keep it simple and link a few resources down there if you want to better customize the solution for your needs.

# Getting started
## Router setup
The most important thing here is to make sure that your home router is set up properly to accept the [VPN/WireGuard](https://www.wireguard.com/) traffic. We’ll use the default VPN configuration of your router to simplify things, but you could totally set up an internal [WireGuard server](https://www.wireguard.com/) and get your router forward a specific port ( [Port Forwarding](https://en.wikipedia.org/wiki/Port_forwarding)) to this server. Before I had the opportunity to test out [OPNSense](https://opnsense.org/), this is the exact setup that I had, and it was running off a [Raspberry Pi](https://amzn.to/3Q0YqLv).

That being said, if your router doesn’t have [WireGuard](https://www.wireguard.com/) built-in, you could check if you can flash it with the latest version of [OpenWRT](https://openwrt.org/). We won’t go through the process here but don’t worry, there’s a ton of guides available already, and we’ll make sure to link a few below.

## Initial OpenWRT configuration
Only follow these steps if you just flashed your router with [OpenWRT](https://openwrt.org/). If you’ve got it running already, you don’t need to do this.

1. Open your favourite browser ( [please don’t use IE, it’s EoL](https://blogs.windows.com/windowsexperience/2022/06/15/internet-explorer-11-has-retired-and-is-officially-out-of-support-what-you-need-to-know/))
2. Click the login button. If you haven’t changed the default admin credentials — shame on you! Once the status page loads, you’ll see a message asking you to change those credentials, please do.
3. Click on System and then Administration
4. On the Router Password page, set a secure password. If you need help picking something up, I recommend that you use a password manager — like 1Password
5. On the same page, make sure to set the SSH access to LAN only

## Install WireGuard
For this part you’ll need to SSH into your router and install the required packages. Just keep in mind that rebooting your router will temporarily drop all traffic on your network (I know, you’ve probably guessed that already)

```bash
# install required packages
opkg update
opkg install luci-proto-wireguard luci-app-wireguard wireguard kmod-wireguard wireguard-tools
# reboot your router
reboot
```

## Add firewall rules
In order to allow for the VPN traffic to go through, we’ll need to poke some holes in the [OpenWRT](https://openwrt.org/) (router)’s firewall. Be careful about what you do here, and you should always make sure that you only open up what’s necessary and restrict the traffic going in and out of your VPN connection. That being said, most people can blindly trust what’s being shown here (you should never blind trust any security things)

Go to LuCI and head to Network -> Firewall -> Port Forwards and create a new rule using the following input:
- Name: WireGuard
- Protocol: UDP
- External Zone: WAN
- External Port: 1234
- Internal Zone: LAN
- nternal IP Address: <The IP address of your device/router (i.e. 192.168.1.1)>
- Internal Port: 1234 <This is the mapped port to your internal Wireguard instance>

When you’re done, click Add -> Save -> Apply. This will allow your VPN/WireGuard clients (i.e. your phone, laptop, etc.) to connect back to your router from the internet, but only by exposing the VPN service.

## Public/private key generation
Here we’ll need to create a key pair that will be used to encrypt the connection between your device(s) and your [WireGuard/VPN](https://www.wireguard.com/) server. Please keep them safe as they’re the only thing that protects you from anarchy (jokes aside, make sure that you keep your private key hidden/safe at all cost)

To do so, [SSH](https://docs.microsoft.com/en-us/windows-hardware/manufacture/desktop/factoryos/connect-using-ssh?view=windows-11) into your router (if not already) and run the following:

```bash
umask 077 && wg genkey > privkey
cat privkey | wg pubkey > pubkey

cat /root/pubkey # this will print your public key, keep it safe
cat /root/privkey # this will print your private key, keep it safe
```

This creates two (2) files in the /root directory of your router, the public key (pub key) and its private key (privkey). You should securely transfer it to your VPN clients (again — these are the devices that you want to connect back to your home network) because we’ll need it when setting up the VPN connection. Also, make sure to copy the private key to your clipboard because we’ll need it in the next step.

## Setting up the WireGuard interface
1. Go into LuCI and head to Network -> Interfaces -> Add New Interface
2. Set the name of the new interface to be wg0
3. Set the protocol to be WireGuard VPN and click Submit
4. Paste the private key (privkey) you got from the previous step into the Private key field
5. Set the listen port to be what you’ve set as Internal Port in your Firewall configuration (1234 in this example, but please use something else)
6. In the IP Addresses field, type a [CIDR](https://www.techtarget.com/searchnetworking/definition/CIDR) (address range) that you want to use as the local VPN subnet (i.e., 10.14.0.1/24). These will be the addresses assigned to your VPN clients once they’re authenticated.
7. Go to the Firewall Settings tab and assign the interface to your LAN zone (if it’s not automatically ben assigned already). This will allow you to access your local network (LAN) devices when you’re connected to your VPN. If you wanted to keep some devices separate (and you should!), you can create another firewall zone specifically for the WireGuard Interface and point it to another VLAN. Once you’re done, click Save -> Apply

## Dynamic DNS (DDNS)
As most ISP (Internet Service Providers) don’t provide us with dedicated IPs, the last thing you’d want to happen is probably that you find out that your public IP has changed when you really needed to connect back to your home network. For this specific reason, you should set up a Dynamic DNS (DDNS) entry that will track changes to your public address automagically so that you don’t have to worry about it anymore. Christian Hofer has created a really good step by step guide on how to achieve this on OpenWRT already, so I won’t be re-inventing the wheel here, just go see [his blog](https://medium.com/@_chriz_/free-dynamic-dns-service-provider-configuration-on-an-openwrt-lede-access-point-f24c62b4e32d).

## Client configuration
Setting up clients will be pretty much the same for all operating systems. Just make sure that you download the WireGuard app that is designed for your device.

First, download the WireGuard app from the [Google Play Store](https://www.google.com/url?cad=rja&cd=&esrc=s&q=&rct=j&sa=t&source=web&uact=8&url=https%3A%2F%2Fplay.google.com%2Fstore%2Fapps%2Fdetails%3Fid%3Dcom.wireguard.android%26hl%3Den_CA%26gl%3DUS&usg=AOvVaw0HrmWoi2ulUd259_c5G3nU&ved=2ahUKEwii89Dhy9_5AhXUEFkFHcdMDFAQFnoECAcQAQ)/ [Apple App Store](https://www.google.com/url?cad=rja&cd=&esrc=s&q=&rct=j&sa=t&source=web&uact=8&url=https%3A%2F%2Fapps.apple.com%2Fus%2Fapp%2Fwireguard%2Fid1441195209&usg=AOvVaw3xLxdqeOrvzOe9yPDcgU9_&ved=2ahUKEwii89Dhy9_5AhXUEFkFHcdMDFAQFnoECAoQAQ) and open it.

1. Tap the plus icon and go to Create from scratch
2. Make up a name for your VPN connection
3. Tap Generate to generate yourself a public and private key
4. In the Adresses field, you’ll assign an IP address that is part of the WireGuard local subnet that we decided on earlier (i.e., 10.14.0.3/32)
5. Leave the Listen out and MTU fields empty unless you’ve tweaked your WireGuard configuration yourself
6. In the DNS servers field, you’d probably want to use your home DNS server (which is probably your router’s gateway — something like 192.168.1.1) or you could give a shot at NextDNS or Cloudflare (1.1.1.1)
7. Tap Add Peer
8. Paste the public key from your /root/ directory of your router
9. Leave the Pre-shared key field blank
10. In the Allowed IPs field, type 0.0.0.0/0,::0 (this will redirect all the traffic to your VPN. If you want to use split VPN tunnelling, only specify the IP addresses/CIDRs that you want to use your VPN for). You should still add the IPv6 redirect even if you aren’t using IPv6, as this will stop your devices from leaking data when connected to IPv6 enabled websites (and there’s a lot of them out there)
11. In the Endpoint field, type the public (WAN) IP address or domain name/DDNS of your router, followed by a colon and the public port number that we decided on earlier. It should look similar to this: 143.56.75.23:1234 or myvpn.adomain.tld:1234
12. In the persistent Keepalive field, type 25
13. Save the connection

# Recap
You did it, good job! You’ve set up a secure connection back to your home network that you can use when you’re on the go, from anywhere in the world. [You can even use it when browsing of public wifis](https://www.tomsguide.com/news/why-you-need-to-use-a-vpn-on-public-wi-fi), so that you keep your traffic private and prevent malicious people from eyeing on you.

We only scratched the surface of what’s possible here, but one of the next steps that you could explore is how you could restrict your VPN traffic to only specific network locations/devices, so that you don’t end up exposing everything to your VPN clients. But again, it’s a good start!

# References
- [https://medium.com/@_chriz_/free-dynamic-dns-service-provider-configuration-on-an-openwrt-lede-access-point-f24c62b4e32d](https://medium.com/@_chriz_/free-dynamic-dns-service-provider-configuration-on-an-openwrt-lede-access-point-f24c62b4e32d)
- [https://www.reddit.com/r/openwrt/comments/bahhua/openwrt_wireguard_vpn_server_tutorial/](https://www.reddit.com/r/openwrt/comments/bahhua/openwrt_wireguard_vpn_server_tutorial/)
- [https://openwrt.org](https://openwrt.org/)
- [WireGuard: fast, modern, secure VPN tunnel](https://www.wireguard.com/)