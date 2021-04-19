# DMZ & Network Hardening Tutorial

This tutorial will cover setting up a DMZ architecture as well as some other network security controls.
Security controls are most effective baked into whatever they are being applied to .. network infrastructure is no different.
While network automation makes it possible to optimize configs very quickly; it is more efficient to just get it right initially.

The topology layout consists of three major zones .. Internal, DMZ, & External.
Think of Internal and DMZ together when reffered to as our private network.
The different between the two is that Internal facilities information the public should access .. that's the DMZ's purpose.
This design strategy is a perfect example if signifcantly increasing security through network segmentation.

Topics to cover:
- Switchport assignment hardening
- Server Setup
- DCHP Snooping
- Arp Inspection
- Firewall configuration - DMZ setup with static routes
- Login / SSH setup
- Access Control Lists (ACL's)

If some of these terms seem foreign, look them up and at least know their basic purpose in a network. 
An extensive in depth knowledge is not required to at least start configuring and see how these protocols work in action.   

\# Add instructions to open up the file \#

If devices have passwords my defaults are:
user: admin
password: cisco
enable: class

## Quick Tips
- The question mark can be used for ANY positional parameter to see available command options
  
- Ctrl + a  ->  Moves cursor to the beginning of line
- Ctrl + e  ->  Moves cursor to the end of the line

## Getting started
Base configuratons (Environment, Vlan's, IP addressing, inter-vlan to static routing) are already set up except the firewall.
These steps are covered in my last tutorial basic practial \# Add link to first network tutorial \#.

## Port Security
Lets start out by assigning end hosts to appropriate vlan & securing physical interfaces.
The vlan design is relatively simply with worker, technician, & separate vlans for various servers.
This helps create segmentation as well as access control when ACL's are applied.

My recommended strategry is configure the trunk & access ports in separate ranges.
Take care of all the uniform commands that apply to all interfaces in that range; then apply interface specific commands like switchport access vlan [vlan #].
Considering this tutorial is absent of trunks since there is only a single switch in the LAN access is only needed.

An ideal approach:
- Assign unused ports to Black_Hole vlan & shut them off
- select all access interfaces & assign uniform commands
- assign interfaces corresponding vlans

On Internal:

`conf t
int ran f0/4-19, g0/1-2
switchport access vlan 999
shut
int ran f0/1-23, g0/1-2
switchport mode access
switchport port-security
switchport port-security mac-address sticky
int f0/1
switchport access vlan 20
int ran f0/2-3
switchport access vlan 10
int f0/20
switchport access vlan 30
int f0/21
switchport access vlan 40
int ran f0/22-23
switchport access vlan 50
end
wr`

On External:

`conf t
int ran f0/4-23, g0/1-2
switchport access vlan 999
shut
int ran f0/1-23, g0/1-2
switchport mode access
switchport port-security
switchport port-security mac-address sticky
int f0/1
switchport access vlan 80
int ran f0/2-3
switchport access vlan 70
end
wr`

On DMZ:

`conf t
int ran f0/4-23, g0/1-2
switchport access vlan 999
shut
int ran f0/1-23, g0/1-2
switchport mode access
switchport port-security
switchport port-security mac-address sticky
int ran f0/1-3, f0/24
switchport access vlan 50
end
wr`


At this point the interfaces should be assigned to their corresponding vlan or shut down.
Each interface can only have a single mac-address tied to it or a security violation will shut down the port.

verify configs:
----------------

`show run
show ip interface brief
show vlan brief
show port security`


## Internal server setup

To configure servers in packet tracer simply open the server & click on the services tab.
Take notice that all unused services are turned off .. which a common security strategy is disable anything unused.
Before configuring any of these services make sure they are toggled on.

DHCP
=====

Now lets set up DHCP to dynamically sign IP addresses to end hosts.
The dhcp addressing is deternmined by the IP table .. lets add a pool for each vlan.

On DHCP server:

Click on services tab .. go to DHCP and fill out the following layout:

Pool                Default Gateway         Start IP        Subnet Mask         Max User        
Name 
------------------------------------------------------------------------------------------
serverPool          0.0.0.0                 192.168.30.0    255.255.255.252         0
Worker Pool         192.168.10.1            192.168.10.2    255.255.255.248         2
Technician Pool     192.168.20.1            192.168.20.2    255.255.255.252         1


When dhcp is configured through a server, the interface vlans should be assigned a helper IP to the dhcp server

On Internal:

`conf t
int vlan 10
ip helper-address 192.168.30.2
int vlan 20
ip helper-address 192.168.30.2
end
wr`

At this point DHCP should be configurable on the end hosts.
Simply click on the end hosts, click on config tab and select FastEthernet0 or click on desktop tab and ip configuration; then switch from static to dhcp


For external it will be configured on the L3 switch instead of a server; which is remarkably similar to a router DHCP config.

On External:

`conf t
ip dhcp excluded-address 10.0.70.1
ip dhcp excluded-address 10.0.80.1
ip dhcp pool WORKERS
network 10.0.70.0 255.255.255.248
default-router 10.0.70.1
ip dhcp pool TECHNICIAN
network 10.0.80.0 255.255.255.252
default-router 10.0.80.1
end
wr`

verify configs:
----------------
`sh run
sh ip dhcp binding
sh ip dhcp pool`

Now the external hosts are ready to be assigned IP's.


Internal FTP
=============

First open ftp server, delete default username & use the following layout:

Username        Password        Permission
--------------------------------------------
admin           cisco           RWDNL
user            cisco           RL


Obviously in an actual scenario significantly stronger passwords should be user.
For the time being lets ignore password security & just focus on the protocols.

verify configs:
----------------
- open command prompt on a end host in Internal zone
- run command `ftp 192.168.40.2`
- supply user & password (admin , cisco)
- run `dir` to see whats available
- run `get [name of file]` to retrieve a file


NTP & Syslog
=============

NTP & Syslog will be configured on Internal, which will provide logging auditability.
NTP will synchronize system clocks of the network devices in the private network.
Syslog will utilize NTP to facilitate log messages to a server for all devices in the private network.
NTP should be configured first .. considering Syslog would rendered useless without it.


On NTP server:
---------------
Enable Authentication
Key:        1
Password:   cisco


On Syslog server:
------------------
Confirm service is activated


On Internal:
-----------------------------
Normally timestamps need to be set but that was already taken care of in base configurations for all devices.
No need to configure but here are the commands for reference:

`service timestamps log datetime msec
service timestamps debug datetime msec`

Now what still needs to be done:

`conf t
ntp server 192.168.50.2
ntp authenticate
ntp authentication-key 1 md5 cisco
ntp trusted-key 1
ntp update-calendar
logging 192.168.50.3
logging on
logging userinfo
logging trap
end
wr
`

verify configs:
----------------

`show clock
show ntp associations
show ntp status
show logging
`


## External server setup
Email
======

In services tab .. confirm  email service is on, set the domain-name to internal.net, & add the following:

User        Password
---------------------
admin       cisco 
user        cisco

verify configs:
----------------
- Send a email


HTTP
=====

Confirm service is enabled for http/https

verify configs:
----------------
- Open a browser on a end host in Internal or External zoness .. connect http://172.16.50.4


External FTP
=============

Refer to the Internal FTP configuration above

verify configs:
----------------
- Same method as Internal FTP

Before moving on to configuring the DMZ lets a few more layer 2 LAN security controls 


## DHCP Snooping 

This feauture monitors a facilitates messages for a DHCP server. 
It helps identify anomalies in the dhcp DORA (Discover, Offer, Request, Acknowledgement) request process & drops improper requests.
This prevents attackters from modifying dhcp traffic to manipulate it's dynamic abilities to gain network access.
The only ports that should be trusted are connection from the switch to the DHCP server and switch to switch connections.

On Internal:
-------------

`conf t
ip dhcp snooping
ip dhcp snooping vlan 10,20
ip dhcp snooping verify mac-address
int f0/20
ip dhcp snooping trust
ip dhcp snooping limit rate 100
int ran f0/1-19, f0/21-23, g0/1-2
ip dhcp snooping limit rate 20
end
wr
`

The above configuration not only sets the server port as trusted .. but also limits the rate of dhcp traffic allowed through the interfaces.

On External:
-------------

`conf t
ip dhcp snooping
ip dhcp snooping vlan 70,80
ip dhcp snooping verify mac-address
int ran f0/1-23, g0/1-2
ip dhcp snooping limit rate 20
end
wr`

verify configs:
----------------
`sh ip dhcp snooping`


## ARP Inspection

This feature is similar in nature & dependent on DHCP Snooping, but instead monitors ARP traffic.
For those unfamiliar, ARP (Address Resolution Protocol) is used to query the switch for the layer 3 IP address based on MAC table.
Its commonly referred to as a layer 2 to layer 3 address mapping protocol.
Without ARP, DHCP would assign an address but the host would be clueless on how to exit the internal network to the internet.
ARP Inspection prevents gratuituous arp requests, which are manually requested rather than a result of the DHCP DORA process.
This prevents request modification and protects the ARP table from being poisoned, packets with invalid MAC to IP bindings will be discarded.

On Internal:
-------------

`conf t
ip arp inspection vlan 10,20
ip arp inspection validate src-mac
end
wr`

On External:
-------------

`conf t
ip arp inspection vlan 70,80
ip arp inspection validate src-mac
end
wr`

The rate limit can also be set like DHCP Snooping .. by default it's 15 which is fine for this network.

verify configs:
----------------
`sh ip dhcp snooping`


## DMZ firewall configuration

Now it's finally time to link the DMZ to the internal zone by configuring the ASA Firewall.
First step is to remove any undesired default settings, then configure the rest prior.
The model firewall is a Cisco ASA 5505, which operaties on a layer 2 vlan setup so an SVI (Switch Virtual Interface) is required for IP assignment like a Layer 3 switch.

On Firewall:
`conf t
no dhcpd enable inside
no dhcpd address 192.168.1.5-192.168.1.36 inside
no dhcpd auto_config outside
telnet timeout 1
int vlan 1
ip addr 192.168.1.2 255.255.255.252
int vlan 2
nameif dmz
ip addr 172.16.50.1 255.255.255.248
security-level 70
no forward interface vlan 1
int vlan 3
nameif outside
ip addr 209.165.200.2 255.255.255.252
int vlan 4
exit
route inside 192.168.0.0 255.255.0.0 192.168.1.1
route outside 0.0.0.0 0.0.0.0 209.165.200.1
object network inside-net
subnet 192.168.0.0 255.255.0.0
nat (inside,outside) dynamic interface
int e0/0
switchport access vlan 1
no shut
int e0/1
switchport access vlan 2
no shut
int e0/2
switchport access vlan 3
no shut
int e0/3
switchport access vlan 4
shut
int e0/4
switchport access vlan 4
shut
int e0/5
switchport access vlan 4
shut
int e0/6
switchport access vlan 4
shut
int e0/7
switchport access vlan 4
shut
exit
class-map CMAP
match default-inspection-traffic
exit
policy-map PMAP
class CMAP
inspect dns
inspect ftp
inspect http
inspect icmp
exit
service-policy PMAP global 
end
write memory`

The above configurtion removes undesired defaults, configures SVI interfaces (black_hole for unused interfaces)
, assigns them to a switchport, & disable any usued interfaces (Packet Tracer firewall unfortunantly absent of access interface ranges).
Then makes a class map that is linked to a policy-map which finally links to the global service-policy.

At this point Internal should be able to communicate any zone while DMZ & External can communicate with each other but are isolate from the Internal zone.
It becomes quite obvious why this is an ideal approach to secure architecture, though it is far from fool proof. 

A well crafted block of javascript or executables initiated through websites, emails, etc. 
This can infect the users session which can blend their traffic in the user stream to bypass the firewall.
Making the attack seem as part of an accepted outgoing connection rather than trying to infiltrate with their own inbound connection.
Thats why end user training is among the most critical aspects of the attack surface, lack of understanding can result in activity that has potential to bypass layers of security controls.

verify configs:
----------------
`sh run
sh ip
sh int ip br
sh nat
sh route
`


## Login/SSH

Normally for larger networks I would recommend utilzing AAA for remote authentication & use local authentication as a backup if the remote server fails.
Considering were only going to harden 3 devices its not much of a concern .. if connectivity issues occur they still would have to be accessed physically reguardless.

on Internal & DMZ:

`conf t
username admin privilege 15 secret cisco
enable secret class
login block-for 300 attempts 5 within 120   # <= Login commands unavailable on DMZ L2 switch
login on-success log
login on-failure log
crypto key generate rsa                     # <= hit enter, select biggest key size, hit enter again (Firewall already has key-pair)
line con 0
privilege level 15
login local
exit
line vty 0 15
privilege level 15
login local
end
wr`

On Firewall:

`conf t
username admin password ciscoasapassword encrypted
enable password class
ssh 192.168.20.2 255.255.255.255 inside
crypto key generate rsa modulus 2048      # <= answer yes when prompted
end
write memory`


verify configs:
----------------
`sh run
sh login
sh ip ssh`

Despite the configuration SSH still might not work on the ASA in packet tracer .. they are a newer feature and still have some buggy issues & less developed than the routers & switches.

While not available on packet tracer I would recommended disabling telnet if & enforcing ssh only.

## Access Control Lists (ACL's)

After lets work on the internal zone as a second layer of protect if the firewall is breached, to isolate internal vlans based on least privilege access control, & for protecting the remote access capabilities of our network device VTY lines.

On Internal:

`conf t
ip access-list standard SyslogNTP
deny any
exit
int vlan 50
ip access-group SyslogNTP in
exit
end
ip access-list extended SSH
permit tcp host 192.168.20.2 any eq 22
exit
line vty 0 15
access-class SSH in
wr`

verify configs:
----------------
`sh run
sh access-lists`

The above sets the NTP / Syslog Vlan to be unaccessable to end users and SSH login only available to the Technician.

Thats all for this tutorial .. feel free to mess around with ping / traceroute testing in the environment between the different zone combinations.