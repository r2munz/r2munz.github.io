---
title: "All the roads lead to Rome - How I routed my home network"
date: 2020-03-18
layout: post
---

Moving my first steps within system administration, I had to figure out how to deal with networking and all the culprits that came across with it. I can still remember touching (by mistake) my interface gateway definition under an ancient version of Windows. Internet works no more, and shivers all over the place.

So I have started to dig up slowly in what is a packet, how it is flowing, getting a basic idea of the OSI model. After that, the best way to learn by doing was... messing up with my own network. Configuring it, misconfiguring it, breaking things multiple times in order to fix them and understand how things *should* be done in order to make it work.

For this purpose, having an ISP provided commodity router isn't the best place to start with. After getting root access to it (this time, wihtout bricking it) I quickly realized that its features and functionalities were pretty limited, constraining my ability to experiment with new configurations and settings. So I have ditched my Technicolor AGTHP DGA 4132, disabled most of the services running on top of it, and use it as a simple modem box to be able to enstablish a PPPoE connection on a DSL cable with a prosumer device.

The current cooking recipe includes the following networking devices:

* Ubiquiti ER4
* Ubiquiti ES8
* Ubiquiti HD Nano AP
* Teltonika RUT 240
* 12 Ports patch panel (with raimbow colored patch cables, for the joy of my little son)
* RaspberryPi 4 4GB

The coolest thing of this setup is that I have managed to squeeze everything (with a decent amount of work) in a nice small form factor 10" rack cabinet. This fact, other than reducing drastically the cables-related mess of a small network and hiding most of the caos behind a thin metal wall, will give me back the feeling of having an *enterprise* grade datacenter (and yes, I can hear your laughs).

![10" rack](https://lh3.googleusercontent.com/pw/AJFCJaXIhIpO2ZiHFV0pDvyKJbZZU8YlATeKidc2ncgsgd05VGQswAERdYiOKhwPS2r3rK2gT9MqnHeygCqkFUFtaUZryz4J-6SX-RD7V18CR1Pqty7Uy8DJ=w1920-h1080)


These are the features I have achieved in my network so far:

1. [Segment my LAN into multiple networks by using VLANs](#vlans)
1. [Use my own DNS, configure my own DHCP server, have a grip on the Firewall rules sitting at the edge of my network](#basic-services) 
1. [Be able to have an automatic failover WAN link (particularly useful when working remotely)](#wan-failover)
1. [Be able to selectively route traffic based on network policies (PBR)](#pbr)
1. [Put an OpenVPN server at the edge of my network](#openvpn-server)

### VLANs

Getting to understand Virtual LAN packet tagging is fundamental to reduce the number of cables and devices that are sitting in your network. In practice, you can segment your network virtually rather than phisically. Without complicating my life to the next level (assuming I would be extraparanoid on my home network security), I haven't used the VLAN id = 1 of my primary LAN router interface (Eth2) for management only. Instead, this is the current topology:

* **VLAN id 1** - 192.168.100.0/24: *Home Network*. Almost all my devices are sitting in this network. DNS is provided by dnsmasq running on the Raspberry Pi, which has a [dnscrypt][dnscrypt] overlay and few filters to avoid advertisements and malware sites. DHCP is served by the ER4
* **VLAN id 101** - 192.168.101.0/24: *Lab Network*. Devices in this network don't have internet access by default. DNS is provided by a dedicated virtual machine in the lab running Fedora Server with [bind][bind].  Bind  has been chosen to serve the DNS lab network in order to overcome the wildcard limitation provided by dnsmasq, which limits down the functionality of OpenShift, making it practically unusable. DHCP is served by the ER4, which is able to statically assign IP addresses according to the physical address of the network interfaces. Few exceptions are made on the firewall rules on the basis of network addresses to allow selected devices out on the internet.
* **VLAN id 102** - 192.168.102.0/24: *Guest/IOT Network*. Devices in this network are isolated from each other at the Access Point level. They cannot access the Home Network whilst the Home network can have access to them. Their bandwidth is limited down to a rate which is useful for checking the email and send small amount of data.
* **VLAN id 103** - 192.168.103.0/24: *VPN Network*. This network is specially designed with privacy requirements. All the devices sitting in this network are tunneled via a Virtual Private Network provided by a different vendor from my ISP. The profile normally applied acquires a Public IP located in Zurich (CH), but the list of server I could choose from is ranging worldwide locations.

Reading about this topology, one of the first question you might ask would be: 

*What's wrong in assigning the 192.168.1.0/24 network range to your home network, as most of the commercial router would do for you?*

The answer is rather [simple][network-conflicts]. 

The first time I wanted to configure my own OpenVPN instance I was sitting on a default 192.168.1.0/24 network. Out in the wild, this is the most common configuration you will find. Which means, if you are sitting at a not-so-nerdy friend's place or a casual coffee shop and you forgot a file at home, you'd like to connect to your network to pick it up from your file server. Then things will start to complicate. Your homey VPN server will try to push the 192.168.1.0/24 route to your client, which, accidentally, already has it in its routing table due to your friends router DCHP settings. Will packet be able to be routed to your cosy home via a virtual private tunnel? 

**Never** 

Hence, if you want to have access to your home remotely (and be successful), you'd better stay away from that network range. Moving next:

*Why do you want to have a Lab environment?*

Having a disconnected environment is good for many testing purposes. My lab environment cosists mainy in a beefy workstation equipped with [ProxMox][proxmox] hypervisor where I can spin up multiple VMs at my convenience. I'll cover these practices in some other posts.

*What about the IoT network?*

Our homes are nowadays smarter and smarter. Which means, you'll have a quite a number of small headless devicess constantly connected to the internet, exchanging data with random cloud servers. Let's suppose one of the firm of these devices is not so keen in pushing security updates to their appliances, maybe you are fine with the idea of a remote attacker taking over your washing machine, but would you really like him/her to go and start to scan your home network looking for other security flaws? 

*A VPN routed network, seriously?*

Well, this is one of my favourite features. First of all because you can potentially use the internet as it was meant to be: freely and with no privacy controls imposed by ISP or gonvernment/corporations overheads. Secondly (maybe more importantly for my taste), because it give me the opportunity to understand the concept of [Policy Based Routing](#pbr)

### Basic Services

#### DHCP

DCHP service lift most of a network admin overhead with a simple managment of the available IPs to the network. Practically, on each network there is going to be a dedicated server which is hopefully properly configured. ER4 is internally leveraging ISC DHCP daemon (DHCPD), however, it can be [configured][dnsmasq-on-er4] to use dnsmasq to serve DHCP as well. The port used is the standard one: 67/UDP. One of the interesting feature of DHCP server is that it can statically assign IPs based on the physical address of the client's interface requesting it. Practically, in order to configure a DHCP server on a given ER4 shared network name with an IP pool ranging from `IP_START` to `IP_END` in a `SUBNET` and to assign a static `HOST_IP` to a specific host based on its `HOST_MAC_ADDRESS` , these commands should be issued on the terminal console:

```
configure

set service dhcp-server shared-network-name <DHCP_SHARED_NETWORK_NAME> subnet <SUBNET> start <IP_START> stop <IP_END>

set service dhcp-server shared-network-name <DHCP_SHARED_NETWORK_NAME> subnet <SUBNET> static-mapping <HOST_ALIAS> ip-address <HOST_IP>

set service dhcp-server shared-network-name <DHCP_SHARED_NETWORK_NAME> subnet <SUBNET> static-mapping <HOST_ALIAS> mac-address <HOST_MAC_ADDRESS>

commit; save
```

Common options, like setting PXE boot server on the LAN (i.e. 66, 67), are configurable under specific option names (`bootfile-name` and `tftp-server-name`, respectively), whilst other custom options might be configured [manually][er-dhcp-custom-options], too.

#### DNS

I decided to delegate DNS serving to my little and beloved RasPi. ER4 could serve the purpose as well, but using dnscrypt on it becomes clunky and difficult to [maintain][dnscrypt-edgerouter]. On the RaspPi instead, dnsproxy runs as a breeze and it is quite straightforward to update your [blacklist][dnscrypt-utils] with a daily cronjob. Dnscrypt is the right choice if you don't like to be profiled by your ISP. Your DNS queries will be always encrypted and your privacy will be therefore guaranteed. On top of that, dnscrypt blacklist support the use of wildcards, which makes them really handy to use. Using blacklists is quite useful if you want your network to be protected from malware sites, ads, aldult contents streaming on your kids devices and so on...

As a general approach I have decided to delegate dnsmasq to resolve all the internal names for my network, looking up on the RasPi `/etc/hosts` table, and forward all the rest to dnscrypt. 

#### Firewall Rules

Understanding how the firewall rules are applied to the EdgeRouter interfaces is of utmost importance if you want to master them. In order to understand this *direction* concept, this diagram comes to rescue.

![layman-diagram](https://lh3.googleusercontent.com/Wy9xtF-BuOLHKYheLOV6R2BF34kTIsW1KIrKRpWcHQ967APPN2lmsXQcyMapJrwvqFXp_dnJNjucY7oxApGF4PrA11G5f961d9Robevja2-oywhSE66o_tJvfFgWo2GgMMkySo4uE5c=w2400)


You will be able to apply your ruleset to a certain interface in three different directions:

* IN: this is probably the most interesting direction as packets coming FROM the selected interface will be filtered accordingly 
* LOCAL: this is a special direction as packets coming FROM the interface TOWARDS the local router services (SSH, DNS, DHCP, GUI...) will be filtered accordingly
* OUT: this direction is rarely used as filtering packets TO the selected interface can be normally achieved by filtering in the IN direction from more suitable interfaces (i.e WAN)


Firewall roulesets are *ordered list* of rules, which means that the relative order in which you place these roules is important. The firewall in fact, will read these rules in a sequential order and apply them to packets accordingly.

*Name* rules are the ones that are filtering your packets according to some defined criteria based on sources, destinations, ports or protocols.
In my network, a bunch of name rouleset are worth mentioning:

* WAN_LOCAL: this rouleset is applied from all the interfaces that are public facing (pppoe, eth1.10, vtun1) towards the local services of the router. Its default action is *drop* followed by Allow Enstablish/Related, Drop Invalid, OpenVPN DNAT on 443 port. From the router console:

```
configure

set firewall name WAN_LOCAL default-action drop
set firewall name WAN_LOCAL description 'WAN to router'

set firewall name WAN_LOCAL rule 10 action accept
set firewall name WAN_LOCAL rule 10 description 'Allow established/related'
set firewall name WAN_LOCAL rule 10 state established enable
set firewall name WAN_LOCAL rule 10 state related enable

set firewall name WAN_LOCAL rule 20 action drop
set firewall name WAN_LOCAL rule 20 description 'Drop invalid state'
set firewall name WAN_LOCAL rule 20 state invalid enable

set firewall name WAN_LOCAL rule 30 action accept
set firewall name WAN_LOCAL rule 30 description 'openvpn'
set firewall name WAN_LOCAL rule 30 protocol udp
set firewall name WAN_LOCAL rule 30 destination port 443

commit; save
```

* WAN_IN: this rouleset is applied from all the interfaces that are public facing (pppoe, eth1.10, vtun1) towards the other interfaces of the router. Its default action is *drop* followed by Allow Enstablish/Related, Drop Invalid. From the router console: 

```
configure

set firewall name WAN_IN default-action drop
set firewall name WAN_IN description 'WAN to internal'

set firewall name WAN_IN rule 10 action accept
set firewall name WAN_IN rule 10 description 'Allow established/related'
set firewall name WAN_IN rule 10 state established enable
set firewall name WAN_IN rule 10 state related enable

set firewall name WAN_IN rule 20 action drop
set firewall name WAN_IN rule 20 description 'Drop invalid state'
set firewall name WAN_IN rule 20 state invalid enable

commit; save
```

* LAB_IN: this rouleset is applied from the lab interface (eth3.101) towards the other interfaces of the router. Its default action is *drop* followed by Allow Enstablish/Related and Allow the LAB_OUT network group to ALL (addresses/ports/protocols). In this way, the lab hosts are segregated in their own network with the exception of the LAB_OUT network group (a well defined address list). Accessing the lab from the other networks is granted by the Allow Enstablish/Related rule. From the router console: 

```
configure

set firewall name LAB_IN default-action drop
set firewall name LAB_IN description 'LAB to internal'

set firewall name LAB_IN rule 10 action accept
set firewall name LAB_IN rule 10 description 'Allow established/related'
set firewall name LAB_IN rule 10 state established enable
set firewall name LAB_IN rule 10 state related enable

set firewall name LAB_IN rule 20 action accept
set firewall name LAB_IN rule 20 description 'Allow LAB_OUT to ALL'
set firewall name LAB_IN rule 20 protocol all
set firewall name LAB_IN rule 20 source group LAB_OUT

commit; save
```

* GUEST_LOCAL: this rouleset is applied from guest/iot interface (eth3.102) towards the local services of the router. Its default action is *drop* followed by specific rules to allow basic DHCP/DNS services and Allow Enstablish/Related. In this way, IoT devices will be able to retrieve a dynamic IP from the router DHCP server and use its internal DNS to resolve the domain names. Accessing other services (like SSH or webGUI) will be denied. Accessing the guest network from the other networks is granted by the Allow Enstablish/Related rule. Moreover, for the devices sitting on the wi-fi network, host to host comunication will be blocked at the AP level. This [port-isolation][port-isolation] feature could be in principle achieved on cabled devices as well, given the fact I have a L2 managed switch, but I haven't look into this option as I don't really need it. From the router console:

```
configure

set firewall name GUEST_LOCAL default-action drop
set firewall name GUEST_LOCAL description 'GUEST to router'

set firewall name GUEST_LOCAL rule 10 action accept
set firewall name GUEST_LOCAL rule 10 description 'DNS'
set firewall name GUEST_LOCAL rule 10 destination port 53
set firewall name GUEST_LOCAL rule 10 protocol tpc_udp

set firewall name GUEST_LOCAL rule 20 action accept
set firewall name GUEST_LOCAL rule 20 description 'DHCP'
set firewall name GUEST_LOCAL rule 20 destination port 67
set firewall name GUEST_LOCAL rule 20 protocol udp

set firewall name GUEST_LOCAL rule 30 action accept
set firewall name GUEST_LOCAL rule 30 description 'Allow established/related'
set firewall name GUEST_LOCAL rule 30 state established enable
set firewall name GUEST_LOCAL rule 30 state related enable

commit; save
```

* GUEST_IN: this rouleset is applied from guest/iot interface (eth3.102) towards the other interfaces of the router. Its default action is *accept* followed by specific rules to allow for exceptions (i.e: my guests should be able to use the printers) and drop all the packets that are trying to access other internal networks. Internal networks are defined in a networkgroup called RFC1918 and includes all private IP subranges. From the router console:

```
configure

set firewall name GUEST_IN default-action accept
set firewall name GUEST_IN description 'GUEST to LAN'

set firewall name GUEST_IN rule 10 action accept
set firewall name GUEST_IN rule 10 description 'Guest to PRINTERS'
set firewall name GUEST_IN rule 10 destination group PRINTERS	
set firewall name GUEST_IN rule 10 protocol all

set firewall name GUEST_IN rule 20 action drop
set firewall name GUEST_IN rule 20 description 'Guest to RFC1918'
set firewall name GUEST_IN rule 20 destination group RFC1918
set firewall name GUEST_IN rule 20 protocol all

commit; save
```

On the other hand, *modify* rules are more subtle and advanced to be used (and they cannot be applied by using the GUI, but only through the CLI or the Config Tree). These rules modifies the trajectory of the packets by allowing them to select and use routing tables that differ from the default router routing table. They are very powerful when you want to set up [load-balancing](#wan-failover) or [policy based routing](#pbr). When setting up these rules, one has to pay special attention when creating new internal interfaces that needs to go out to the internet. If you forget to apply these rules to freshly created interfaces, these won't work as expected and you'll wonder why without being able to find useful solution on google srtaight the way.

#### NAT

NAT, or Network Address Translation, is a core network service which is used in two directions: Source and Destination.

* Source: also called Masquerade, its most common function is to translate the packets source address from private to a public one. In this way, the packets send out to the internet will know their way back to the host which requested them. In my network, two public facing interfaces are currently natted: pppoe and vtun1 being the first ones the link enstablished with my ISP and the second one a VPN link enstablished with an external provider called UNS.
* Destination: this function is also called Port Forwarding and it's useful to expose ports, addresses or network groups to your public ip. As the name is sayng, the router is translating the destination's addresses to reach hosts that are behind the router.


### WAN Failover

Before digging into this section, I want to explain how my cable management obsession got really on the way to set this feature up:
The second IoT router I am using to provide a mobile WAN link can be powered up by POE, a feature that my managed switch currently supports. This means that in principle, I can get rid of some power cables by relying on power being transmitted over the ethernet cable by my switch. In order to achieve this result, I made a smart use of VLAN tagging:
ER4 Eth1 interface has been equipped with a VLAN ID = 10. One of the port of my ES8 is receiving this VLAN ID, untagging it on a second port (powered by POE) which connects to the Eth1 interface of RUT240 and excluding it from all the other ports. In this way, I put my RUT240 upstream from the rest of my networks, so it can provide other basic services on its own (DHCP, DNS, NAT). In this way, taking it to vacation as my mobile router is just a matter of unplugging an ethernet cable: my home won't have WAN failover anymore, but you know, when I am not around I won't care so much...
Clearly, the downside of this configuration is that I am using two switch ports to achieve the purpose.

![port-config](https://lh3.googleusercontent.com/oJXM-0m3Uyv8p4mvaciPLiMhxEYLPHbNjxn9s9Q0IK96fGGQ-poSuo4rXTOzOkZpv8QeDHYL-S_Ycv-PopPWlwphrsUY8kvOYnpy1d_VanystLCZbmLWYQvhmqksJN3mJ9IiD_shwfQ=w2400)

WAN failover is achieved by creating a load balance group in which the two interfaces are defined. The load balance group has a tons of useful options, but I am going to use the failover-only one for this scope. The commands issued to set this feature up are going to be summarized in the [next section][#pbr].

### PBR

Policy based routing is a powerful feature that allows your packets to be re-routed according to some specific network policies. In my setup, I wanted a very specific VLAN to be routed out on the internet via a virtual private tunnel. In this way, all the hosts connected to the network sitting on that VLAN tunnel the usual WAN link and be able to appear having another public address. In this way several geolocalization and privacy restriction can be worked around.
Say you would like to watch the US Netflix catalogue. Without having to install the ovpn profile on all the devices that would like to do that, it is enough to connect these devices to that internal network. In order to set up PBR and WAN failover, these are the commands I have used from a router console:

```
configure

set network group LAN description "Local area networks"
set network group LAN network 192.168.100.0/24
set network group LAN network 192.168.101.0/24
set network group LAN network 192.168.102.0/24
set network group LAN network 10.0.10.0/24

set network group LAN_VPN description "Members of LAN which access UN VPN"
set network group LAN network 192.168.103.0/24

set network group RFC1918 description "Private networks"
set network group RFC1918 network 10.0.0.0/8
set network group RFC1918 network 172.16.0.0/12
set network group RFC1918 network 192.168.0.0/16

set load-balance group LB1 interface eth1.10 failover-only
set load-balance group LB1 interface pppoe0

set protocols static table 10 interface-route 0.0.0.0/0 next-hop-interface pppoe0
set protocols static table 10 description "WAN table"
set protocols static table 20 interface-route 0.0.0.0/0 next-hop-interface vtun1
set protocols static table 20 description "UN-VPN table"

set firewall modify PBR rule 5 action modify
set firewall modify PBR rule 5 description "Route all traffic to RFC1918 netowrks via main table"
set firewall modify PBR rule 5 destination group network-group RFC1918
set firewall modify PBR rule 5 modify table main

set firewall modify PBR rule 10 action modify
set firewall modify PBR rule 10 description "Route traffic from group LAN-VPN through UN-VPN table"
set firewall modify PBR rule 10 destination group address-group LAN-VPN
set firewall modify PBR rule 10 modify table 20

set firewall modify PBR rule 20 action modify
set firewall modify PBR rule 20 description "Route traffic from group LAN through WAN table"
set firewall modify PBR rule 20 destination group address-group LAN
set firewall modify PBR rule 20 modify table 10

set firewall modify PBR rule 100 action modify
set firewall modify PBR rule 100 modify lb-group LB1

set interfaces ethernet eth3 firewall in modify PBR
set interfaces ethernet eth3.101 firewall in modify PBR
set interfaces ethernet eth3.102 firewall in modify PBR
set interfaces ethernet eth3.103 firewall in modify PBR
set interfaces ethernet vtun0 firewall in modify PBR

commit; save
```

### OpenVPN Server 

Setting up an OpenVPN server at the edge of the network is something very convenient. First of all because your router is going to be likely always on, so you don't need to power up a separated device to achieve that role and he is going to take care of the computational effort of encrypting/decrypting the traffic through the tunnel. The most obvious reason though, is that you'll be able to access you local devices and services remotely. This will probably help you get around some stupid password sharing policies set up by streaming companies.
I have created my personal PKI (Private Key Infrastructure) on my RasPi by means of the [easy-rsa][easy-rsa] bundle. So I created my CA (Certificate Authority) through which I can sign, renew and revoke requests. Certificates will be then used to compile ovpn profiles to be installed in the devices that will be allowed to connect to my OpenVPN server.


### Conclusion

If you made it through here, well congratulations! You have my highest appreciation and recognition for bein so patient and interested. We can summarize what I wrote so far with the following, simplified, representation of the whole topology setup.

![network-diagram](https://lh3.googleusercontent.com/IxTN3jFCYBMwRaKSO56eUQCZChl2Osn_fVrJNEBeazanDF8z2Lmhc5-bc85B1eDTtXTaZIXe-MZpzR6M_0PbGKXGdtZB4FmwPSgcupnhnTN0-u8HFXVyK5oazzxO5j5BwggQfoXRKrI=w2400)

[dnscrypt]: https://www.dnscrypt.org/
[bind]: https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/managing_networking_infrastructure_services/assembly_setting-up-and-configuring-a-bind-dns-server_networking-infrastructure-services
[network-conflicts]: https://forums.openvpn.net/viewtopic.php?t=24541
[proxmox]: https://www.proxmox.com/en/proxmox-ve
[dnsmasq-on-er4]: https://help.ui.com/hc/en-us/articles/115002673188
[er-dhcp-custom-options]: https://help.ui.com/hc/en-us/articles/204960074-EdgeRouter-Custom-DHCP-Server-Options
[dnscrypt-edgerouter]: https://github.com/darkgrue/Ubiquiti-DNSCrypt-Proxy-2-Configuration-Scripts
[dnscrypt-utils]: https://github.com/DNSCrypt/dnscrypt-proxy/tree/master/utils/generate-domains-blocklist
[port-isolation]: https://help.ui.com/hc/en-us/articles/360039311974-EdgeSwitch-EdgeSwitch-X-Port-Isolation
[easy-rsa]: https://github.com/OpenVPN/easy-rsa