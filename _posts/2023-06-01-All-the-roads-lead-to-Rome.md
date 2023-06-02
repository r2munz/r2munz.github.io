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

I decided to delegate DNS serving to my little and beloved RasPi. ER4 could serve the purpose as well, but using dnscrypt on it becomes clunky and difficult to [maintain][dnscrypt-edgerouter]. On the RaspPi instead, dnsproxy runs as a breeze and it is quite straightforward to update daily your [blacklist][dnscrypt-utils] with a daily cronjob. Dnscrypt is the right choice if you don't like to be profiled by your ISP. Your DNS queries will be always encrypted and your privacy will be therefore guaranteed. On top of that, dnscrypt blacklist support the use of wildcards, which makes them really handy to use. Using blacklists is quite useful if you want your network to be protected from malware sites, ads, aldult contents streaming on your kids devices and so on...

As a general approach I have decided to delegate dnsmasq to resolve all the internal names for my network, looking up on the RasPi `/etc/hosts` table, and forward all the rest to dnscrypt. 

#### Firewall Rules

Understanding how the firewall rules are applied to the EdgeRouter interfaces is of utmost importance if you want to master them. In order to understand this *direction* concept, this diagram comes to rescue.

![layman-diagram](https://lh3.googleusercontent.com/Wy9xtF-BuOLHKYheLOV6R2BF34kTIsW1KIrKRpWcHQ967APPN2lmsXQcyMapJrwvqFXp_dnJNjucY7oxApGF4PrA11G5f961d9Robevja2-oywhSE66o_tJvfFgWo2GgMMkySo4uE5c=w2400)

You will be able to apply your ruleset to a certain interface in three different directions:

* IN: this is probably the most interesting direction as packets coming FROM the selected interface will be filtered accordingly 
* LOCAL: this is a special direction as packets coming FROM the interface TOWARDS the local router services (SSH, DNS, DHCP, GUI...) will be filtered accordingly
* OUT: this direction is rarely used as filtering packets TO the selected interface can be normally achieved by filtering in the IN direction from more suitable interfaces (i.e WAN)

*Name* rules are the ones that are filtering your packets according to some defined criteria based on sources, targets, ports or protocols. 
On the other hand, *modify* rules are more subtle and advanced to be used (and they cannot be applied by using the GUI, but only through the CLI or the Config Tree). These rules modifies the trajectory of the packets by allowing them to select and use routing tables that differ from the default router routing table. They are very powerful when you want to set up [load-balancing](#wan-failover) or [policy based routing](#pbr). When setting up these rules, one has to pay special attention when creating new internal interfaces that needs to go out in the internet. If you forget to apply these rules to the new interface, this won't work as expected and you'll wonder why without being able to find useful solution on google srtaight the way.


### WAN Failover
### PBR
### OpenVPN Server 

[dnscrypt]: https://www.dnscrypt.org/
[bind]: https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/managing_networking_infrastructure_services/assembly_setting-up-and-configuring-a-bind-dns-server_networking-infrastructure-services
[network-conflicts]: https://forums.openvpn.net/viewtopic.php?t=24541
[proxmox]: https://www.proxmox.com/en/proxmox-ve
[dnsmasq-on-er4]: https://help.ui.com/hc/en-us/articles/115002673188
[er-dhcp-custom-options]: https://help.ui.com/hc/en-us/articles/204960074-EdgeRouter-Custom-DHCP-Server-Options
[dnscrypt-edgerouter]: https://github.com/darkgrue/Ubiquiti-DNSCrypt-Proxy-2-Configuration-Scripts
[dnscrypt-utils]: https://github.com/DNSCrypt/dnscrypt-proxy/tree/master/utils/generate-domains-blocklist