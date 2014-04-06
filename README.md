#Tunlr Clone for Non-SNI Devices#
With the closure of Tunlr and the move to paid subscriptions for Unlocator, options for a working, permanent DNS spoofing/proxy server for geolocation spoofing became either more expensive or harder to find. If you are technically savvy, then the cheapest alternative is to setup a DNS spoofing/proxy server yourself using a VPS provider. There's even a nice guide [here](http://corporate-gadfly.github.io/Tunlr-Clone/). Unfortunately, this will only work for SNI-capable clients -- clients that support a newer TLS extension that sends the subdomain in plaintext during the initial connection handshake. For clients that don't support SNI (like most consoles and dedicated video streamers -- even newer ones), the only way to properly route packets to the proper destination is to provide a unique IP address during DNS spoofing and then routing all traffic that comes in through a certain IP to the designated subdomain for that IP (see [here](http://trick77.com/2014/03/01/tunlr-style-dns-unblocking-pandora-netflix-hulu-et-al/)). Unfortunately, IPv4 IPs are getting scarce and thus more expensive, and any cost-savings from setting up your own server instead of using an existing paid service would be negated if you bought enough public IPs to do this.

So, how can you get around this? By offloading the DNS spoofing to your LAN! IPs on your LAN are, of course, free (assuming you are using a standard router/gateway setup on a residential LAN in the first place). And, in Linux at least, getting more than one IP on the same device is as simple as creating a virtual device and assigning it an IP. Thus, if you create enough IPs for each required subdomain on a Linux server, and run an in-house DNS spoofer that sends each subdomain to a unique LAN IP, then you can forward all of the traffic coming from each **IP** to a unique **port** on your VPS, which in turn can send all traffic coming in on each port to the proper Netflix domain. In doing so, now all of the traffic from your LAN that needs to be geolocated to the US is properly proxied from *any* device, even those that don't support SNI. Basically, this just gets around the lack of port information available to DNS queries and translates internal IPs to external ports -- so you will only need one public US IP.

The gotcha, of course, is that this will only work on your LAN (or on any other LAN with a server running an identical setup), and requires a server on your LAN. You can still run a standard DNS spoofer/proxy server on your VPS as well for SNI-capable devices if you want to use them outside of your LAN. If you don't already have a server on your LAN, then the cheapest option is to get a Raspberry Pi or similar device and set it up accordingly -- but this may push the cost above a reasonable amount (it would take one to two years of streaming to make it worth the purchase over using an existing service). And there is certainly no guarantee that Netflix's CDN won't enable geolocation at some point, rendering this exercise moot (though you could still use the VPS as a VPN I suppose, if that became necessary). If, however, you have a Linux server (or equivalent) running on your LAN already, then this becomes the cheapest Netflix proxying service.

##How To##
It is easiest to do this out of order since services that actually occur earlier in the process depend on the setup of services that occur later.

###Assigning IPs###
You must first assign multiple IP addresses to your LAN server. Many distros allow you to do this through network configuration files, but since there are a lot of IPs it is easiest just to run something like the following from `rc.local` or its equivalent (though this needs to run before the proxy server starts so depending on the distro you may need to think the start-order through):
```bash
# Create interfaces for DNS spoofing
for i in `seq 0 16`; do ifconfig eth0:${i} 192.168.0.$(($i+101)) up; done
```

This will create virtual interfaces eth0:0 to eth0:16, with IPs 192.168.0.101 to 192.168.0.117.
For linux distributions that do not have ifconfig anymore but instead use the ip command this line will do the same thing:
```bash
# Create interfaces for DNS spoofing
for i in `seq 0 16`; do ip addr add 192.168.0.$(($i+101)) dev eth0; done
```

This will assign additional ips to the eth0 interface.
You should check to make sure the additional addresses were created using ifconfig or ip, and check to make sure they work properly by pinging a few of those IPs. Generally, you should assign your server's real IP to be 192.168.0.100 so all IPs are consecutive. Feel free to change the IP range used, but note that you will have to modify the LAN proxy server and DNS spoofer configuration accordingly. Most routers allow certain ranges to be static, so you may want to play within that range to avoid trying to grab an IP already in use by another device.

###The LAN Proxy Server###
You can probably use HTTPS-SNI-Proxy as described in the [Tunlr Clone guide](http://corporate-gadfly.github.io/Tunlr-Clone/), but the documentation is sparse and I'm not sure if you can bind multiple IPs to the same instance and still configure them independently (and since you will need something like 17 IPs for a full Netflix solution, allowing for most known hardware, running 17 instances of HTTPS-SNI-Proxy, all configured differently, is madness). You should definitely use HAProxy instead (and you need to use the latest development version, NOT the stable one!). My HAProxy configuration is inspired by the full haproxy.conf from [here](https://github.com/trick77/tunlr-style-dns-unblocking) (not the poor-man's configuration, which only allows SNI-capable clients to proxy).

Some important notes should accompany this configuration.

1. Replace all instances of 1.2.3.4 with your VPS's public IP.
2. As you can see, I am only proxying Netflix; you can feel free to add other services exactly the same way.
3. You will notice that in the backend sections I am *not* sending server-alive checks. With these enabled it was using quite a bit of bandwidth on my VPS and I certainly don't want Netflix to be suspicious because an IP is sending so many server-alive checks (which would still get forwarded through the VPS!). It is safe to assume that the servers will be available; if not, the connection should just timeout.
4. Make sure you read and understand this configuration before using it! You must change the password (or disable the stat interface entirely if you want), and you may need to tweak things (IPs and port numbers, for example) for your purposes.

###The VPS Proxy Server###
It is wise again to use HAProxy here. Note that my configuration does not allow full non-LAN proxying for SNI-capable clients; in order to do so, you would need to add domains to the `f_sni_catch_all`, `b_sni_catch_all`, `f_sni_catch_all_ssl` and `b_sni_catch_all_ssl` sections (essentially you would merge the poor-man's proxy configuration from [here](https://github.com/trick77/tunlr-style-dns-unblocking) with those sections). Again, the same notes from the LAN proxy server apply here.

###The LAN DNS Spoofer###
The easiest way to spoof DNS entries is to use `dnsmasq` -- the entries you want to spoof are intercepted but the rest of your DNS queries are simply forwarded on to the normal DNS server. Here is the command-line form for dnsmasq (yikes!):
```
dnsmasq -b -D --cache-size=0 -A /www.netflix.com/192.168.0.101 -A /appboot.netflix.com/192.168.0.101 -A /cbp-us.nccp.netflix.com/192.168.0.101 -A /signup.netflix.com/192.168.0.102 -A /api-global.netflix.com/192.168.0.103 -A /movies.netflix.com/192.168.0.104 -A /secure.netflix.com/192.168.0.105 -A /moviecontrol.netflix.com/192.168.0.106 -A /api.netflix.com/192.168.0.107 -A /api-us.netflix.com/192.168.0.108 -A /uiboot.netflix.com/192.168.0.109 -A /cbp.nccp.netflix.com/192.168.0.110 -A /ios.nccp.netflix.com/192.168.0.111 -A /xbox.nccp.netflix.com/192.168.0.112 -A /nccp-nrdp-31.cloud.netflix.net/192.168.0.113 -A /nintendo.nccp.netflix.com/192.168.0.114 -A /playstation.nccp.netflix.com/192.168.0.115 -A /nrdp.nccp.netflix.com/192.168.0.116 -A /android.nccp.netflix.com/192.168.0.117
```
or see the (untested) configuration file, which is certainly cleaner.

###Non-LAN Access###
If you also want to allow SNI-capable clients to geolocate to the US when not on your LAN then you will have to additionally run a nameserver on the VPS (but be careful about running your own publicly accessible DNS server...).

###Restricted Access###
You should probably restrict *all* services on the VPS to just your IP(s) (maybe ssh excepted to avoid getting locked out, but server security is another topic). Certainly it would be wise to use iptables to restrict access to the HAProxy ports (80, 443, 8002-8017, and 9002-9017 in my configuration -- though if you intend to allow SNI clients globally then you can leave 80 and 443 unblocked) and, if running, the DNS server. If your IP fluctuates wildly then this may be more difficult (and you should definitely look up the consequences of using dyndns with iptables -- hint: it won't work without additional work). But, I will leave your server's security up to you.

###Testing###
You can test whether dnsmasq is working properly by performing DNS queries for netflix.com (which should resolve to the real netflix.com IP) and movies.netflix.com (which should resolve to 192.168.0.104 in my configuration) using something like `nslookup` or `host`. Of course you will have to point each client to your LAN server's IP for its DNS resolver -- you can do this individually or perhaps by setting the default nameserver in your router's configuration and letting DHCP do the rest. After that, you should be good to go.

Enjoy!

| File | Description |
| ---------- | ---------- |
| lan-haproxy.cfg | Sample configuration for a LAN proxy that translates multiple IPs to multiple ports for proper non-SNI subdomain forwarding. Requires 17 IP addresses for this specific configuration. |
| vps-haproxy.cfg | Sample configuration for a VPS proxy that forwards packets from different ports to the appropriate Netflix domains. In its current configuration this will only work with an appropriate LAN proxy server; with simple modifications it could also allow SNI-capable clients to use it directly (in concert with a VPS-side DNS spoofer). |
| dnsmasq.conf | A matching dnsmasq configuration for use with lan-haproxy.conf. This is required for in-house DNS spoofing for non-SNI clients. |
| iplist.txt | A list of IP addresses and ports, and where they get forwarded to. This is just to keep track of this confusing mess in case you need to make changes. |
