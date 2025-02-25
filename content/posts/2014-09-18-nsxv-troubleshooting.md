---
title: "NSX vSphere troubleshooting"
description: when things break again !
date: 2014-09-15T12:00:00
tags: ['howto', 'nsx']
---

[Last week](2014-09-14-openvswitch-troubleshooting) we reviewed all the tips & tricks to troubleshoot *Open vSwitch* and *OpenStack Neutron*. *NSX vSphere* (NSX-v) is a different beast, mostly because it leverage *VMware Distributed Switch* (VDS) instead of *Open vSwitch*. As a cheatsheet, I'm gathering all the CLI to troubleshoot it over here, for easy cut & past, some commands are damn long. But wait don't forget the tab completion on our NSX CLI, it's pretty nice ;) But you have to know where to start, hope this helps.

<!-- more -->

### NSX vSphere Web UI

Before jumping into the marvelous world of command lines, as a starter, we'll check the state of the environment from the *vSphere Web Client UI* standpoint.

Authenticate to your web client, and click on `Network & Security > Installation > Management`. 

![][nsxv-controller-status]

You should see a green status for your three controllers.

Next click on `Network & Security > Installation > Host Preparation` and open up each cluster.

![][nsxv-clusters-status]

All the nodes are also green.

Now click on `Network & Security > Installation > Logical Network Preparation` and open up each cluster

![][nsxv-preparation-status]

Each compute node should have a Virtual Tunnel Endpoint (VTEP) vmkernel interface (vmk3 here) with an IP Address assigned to it.

Don't worry if you get any errors, this article is meant to help you troubleshoot the root cause.

### Transport Network

If VXLAN Connectivity isn't operational, I mean if a VM on a VXLAN cannot ping another one on the same logical switch the most common reason is a misconfiguration on the transport network. To check that, SSH to a Compute node and type :

	ping ++netstack=vxlan -d -s 1572 -I vmk3 1.2.3.4

`++netstack=vxlan` instruct the ESXi host to use the VXLAN TCP/IP stack.  
`-d` set Don’t Fragment bit on IPv4 packet  
`-s 1572` set packet size to 1572 to check if MTU is correctly setup up to 1600  
`-I` VXLAN vmkernel interface name  
`1.2.3.4` Destination ESXi host IP Address  

If the ping fails, launch another one without the don't fragment/size argument set

	ping ++netstack=vxlan -I vmk3 1.2.3.4

If this one succeed, it means your MTU isn't correctly set to at least 1600 on your transport network.

If both fails it's a VLAN ID or Uplink misconfiguration. Before going any further you have to make sure that these pings works.

If both succeed, but you still don't have connectivity on the virtual wire, I'll show you, in the Compute node controller connectivity section, how to investigate that using `net-vdl2 -l`.

Note: If you don't know the name of your VXLAN vmkernel you can easily check it, by looking at the configuration of your VDS.

![][nsxv-vds]

But you've also seen that information in the Logical Network Preparation UI above.

### Controller

You can get the IP Address of your Controller by clicking on the VM named `NSX_Controller_<ID>` in the *vSphere Web Client*.

![][nsxv-controller-vm]

To investigate controller issues, SSH to one of your controller VM to use the CLI (login: admin, password: the one set at deployment time).

#### status

	# show control-cluster status
	
	Type                Status                                       Since
	--------------------------------------------------------------------------------
	Join status:        Join complete                                09/14 14:08:46
	Majority status:    Connected to cluster majority                09/18 08:45:16
	Restart status:     This controller can be safely restarted      09/18 08:45:06
	Cluster ID:         b20ddc88-cd62-49ad-b120-572c23108520
	Node UUID:          b20ddc88-cd62-49ad-b120-572c23108520
	
	Role                Configured status   Active status
	--------------------------------------------------------------------------------
	api_provider        enabled             activated
	persistence_server  enabled             activated
	switch_manager      enabled             activated
	logical_manager     enabled             activated
	directory_server    enabled             activated

List all the nodes in the cluster.

	# show control-cluster startup-nodes
	
	192.168.110.201, 192.168.110.202, 192.168.110.203

List the implemented role on your controller, the `Not Configured` for api_provider is normal, it's the NSX-Manager who's published the NSX-v API.

	# show control-cluster roles

	                          Listen-IP  Master?    Last-Changed  Count
	api_provider         Not configured      Yes  09/18 08:45:17      6
	persistence_server              N/A      Yes  09/18 08:45:17      5
	switch_manager            127.0.0.1      Yes  09/18 08:45:17      6
	logical_manager                 N/A      Yes  09/18 08:45:17      6
	directory_server                N/A      Yes  09/18 08:45:17      6

List current connections to your controller.

	# show control-cluster connections

	role                port            listening open conns
	--------------------------------------------------------
	api_provider        api/443         Y         1
	--------------------------------------------------------
	persistence_server  server/2878     Y         2
	                    client/2888     Y         3
	                    election/3888   Y         0
	--------------------------------------------------------
	switch_manager      ovsmgmt/6632    Y         0
	                    openflow/6633   Y         0
	--------------------------------------------------------
	system              cluster/7777    Y         2

Get Controller Statistics

	# show control-cluster core stats

	role                port            listening open conns
	--------------------------------------------------------
	api_provider        api/443         Y         1
	--------------------------------------------------------
	persistence_server  server/2878     Y         2
	                    client/2888     Y         3
	                    election/3888   Y         0
	--------------------------------------------------------
	switch_manager      ovsmgmt/6632    Y         0
	                    openflow/6633   Y         0
	--------------------------------------------------------
	system              cluster/7777    Y         2


#### Controller networking 

	# show network interface
	
	Interface       Address/Netmask     MTU     Admin-Status  Link-Status
	breth0          192.168.110.201/24  1500    UP            UP
	eth0                                1500    UP            UP

	# show network default-gateway
	# show network dns-servers

NTP is mandatory, so make sure it's correctly configured

	# show network ntp-servers
	# show network ntp-status

To troubleshoot controller networking you can also use

	# traceroute <ip_address or dns_name>
	# ping <ip address>
	# ping interface addr <alternate_src_ip> <ip_address>
	# watch network interface breth0 traffic

#### L2 networking troubleshooting

First make sure to connect on the master controller of the virtual network you want to troubleshoot. You can then use the following commands

	# show control-cluster logical-switches vni 5001
	VNI      Controller      BUM-Replication ARP-Proxy Connections VTEPs
	5001     192.168.110.201 Enabled         Enabled   0           0

As you can see above, you should now connect to `192.168.110.201` to troubleshoot VNI 5001.

Let see what we can use on that controller node to get more information on this virtual wire (VNI = 5001).

If you want to check the managemement TCP connection between Controller and ESXi

	# show control-cluster logical-switches connection-table 5001
	Host-IP         Port  ID
	192.168.110.51  17528 2
	192.168.110.52  46026 3
	192.168.210.56  42257 4
	192.168.210.51  30969 5
	192.168.210.57  12127 6
	192.168.210.52  30280 7
	
To see which ESXi instantiate this Logical Switch (LS).

Note: Mac address in the output are the one from the VXLAN vmkernel interface not the one from the Physical uplink.

	# show control-cluster logical-switches vtep-table 5001
	VNI      IP              Segment         MAC               Connection-ID
	5001     192.168.250.54  192.168.250.0   00:50:56:6c:20:30 4
	5001     192.168.250.53  192.168.250.0   00:50:56:60:24:53 6
	5001     192.168.250.52  192.168.250.0   00:50:56:61:23:00 7
	5001     192.168.250.51  192.168.250.0   00:50:56:6b:4b:a4 5
	5001     192.168.150.51  192.168.150.0   00:50:56:60:6a:3a 2
	5001     192.168.150.52  192.168.150.0   00:50:56:6e:5e:e3 3

List Mac addresses on a Logical Switch.

	# show control-cluster logical-switches mac-table 5001
	VNI      MAC               VTEP-IP         Connection-ID
	5001     00:50:56:ae:9b:be 192.168.250.51  5

Same for ARP table on the Logical Switch

	# show control-cluster logical-switches arp-table 5001

List the VNIs on a specific ESXi

	# show control-cluster logical-switches joined-vnis <ESXi_MGT_IP>
	VNI      Controller      BUM-Replication ARP-Proxy Connections VTEPs
	5001     192.168.110.201 Enabled         Enabled   6           6

Shows ESXi VTEP IP/Mac addresses for each joined VNIs

	# show control-cluster logical-switches vtep-records <ESXi_MGT_IP>
	VNI      IP              Segment         MAC               Connection-ID
	5001     192.168.150.51  192.168.150.0   00:50:56:60:6a:3a 2

List all VMs Mac addresses on each VNIs of a specific ESXi with their associated VTEP.

	# show control-cluster logical-switches mac-records <ESXi_MGT_IP>

List all VMs IP/Mac on each each VNIs of a specific ESXi

	# show control-cluster logical-switches arp-records <ESXi_MGT_IP>

#### L3 networking troubleshooting

First you can list all of your logical routers

	# show control-cluster logical-routers instance all
	LR-Id      LR-Name            Hosts[]         Edge-Connection Service-Controller
	1460487509 default+edge-1     192.168.110.51                  192.168.110.201
	                              192.168.110.52
	                              192.168.210.52
	                              192.168.210.51
	                              192.168.210.57
	                              192.168.210.56

You can then use the LR-Id above to get interface details on one instance

	# show control-cluster logical-routers interface-summary 1460487509
	Interface                        Type   Id           IP[]
	570d45550000000b                 vlan   100
	570d45550000000c                 vxlan  5004         10.10.10.1/24
	570d45550000000a                 vxlan  5000


Use the Interface name to get even more details on VXLAN 5004 LIF for example
	
	# show control-cluster logical-routers interface 1460487509 570d45550000000c
	Interface-Name:   570d45550000000c
	Logical-Router-Id:1460487509
	Id:               5004
	Type:             vxlan
	IP:               10.10.10.1/24
	DVS-UUID:         1cec0e50-029c-a921-b6d8-d0fc73e57969
	                  ee660e50-e861-6d04-b4d8-1d462df952bc
	Mac:              02:50:56:8e:21:35
	Mtu:              1500
	Multicast-IP:     0.0.0.1
	Designated-IP:
	Is-Sedimented:    false
	Bridge-Id:
	Bridge-Name:

To get the routing table of your logical router

	# show control-cluster logical-routers routes 1460487509

#### Bridging

To get more information on a all bridge instance hosted on your logical router

	# show control-cluster logical-routers bridges <lr-id> all
	LR-Id       Bridge-Id   Host            Active
	1460487509  1           192.168.110.52  true

And now the Mac address on them

	# show control-cluster logical-routers bridge-mac <lr-id> all
	LR-Id       Bridge-Id   Mac               Vlan-Id Vxlan-Id Port-Id   Source
	1460487509  1           00:50:56:ae:9b:be 0       5000     50331650  vxlan

### Compute Nodes

In the introduction we've seen how to send ping on the transport network. But from a SSH connection to an ESXi node, we can use many more troubleshooting commands. This section will details most of them.

#### VIBs

When you prepare a compute host for NSX-v, vSphere Installation Bundle (VIBs) are automatically installed by the NSX Manager via ESX Agency Manager (EAM). You can check they were correctly installed (output abridged):

	# esxcli software vib get --vibname esx-vxlan
   		...
   		Summary: Vxlan and host tool
   		Description: This package loads module and configures firewall for vxlan networking.
   		...
   		Provides: vxlan = 2.0.0.0-nsx, vdr = 1.0.0.0
   		Maintenance Mode Required: False
   		...

	# esxcli software vib get --vibname esx-vsip
   		...
   		Summary: vsip module
   		Description: This package contains DFW and NetX data and control plane components.
   		...
   		Provides: vsip = 1.0.0-0
   		Maintenance Mode Required: False
   		...

   	# esxcli software vib get --vibname esx-dvfilter-switch-security
	   ...
	   Summary: dvfilter-switch-security module
	   Description: This package contains dvfilter-switch-security module.
	   ...
	   Provides: switchSecurity = 0.1.0.0
	   Maintenance Mode Required: False
	   ...
	
When you remove a host from an NSX Prepared cluster the VIBs will be automatically removed, but you can remove them from the command line :
	
	# esxcli software vib remove -n esx-vsip
	# esxcli software vib remove -n esx-vxlan
	# esxcli software vib remove -n esx-dvfilter-switch-security

You'll then have to reboot your host.

#### Physical Nics

To list all the host physical interface, to check driver and MTU:

	# esxcli network nic list

	Name    PCI Device     Driver  Link  Speed  Duplex  MAC Address         MTU  Description
	------  -------------  ------  ----  -----  ------  -----------------  ----  ------------------	-------------------------
	vmnic0  0000:000:14.0  igb     Up     1000  Full    00:25:90:f4:76:8e  1600  Intel Corporation 	Ethernet Connection I354
	vmnic1  0000:000:14.1  igb     Up     1000  Full    00:25:90:f4:76:8f  1600  Intel Corporation 	Ethernet Connection I354
	vmnic2  0000:000:14.2  igb     Up     1000  Full    00:25:90:f4:76:90  1600  Intel Corporation 	Ethernet Connection I354
	vmnic3  0000:000:14.3  igb     Up     1000  Full    00:25:90:f4:76:91  1600  Intel Corporation 	Ethernet Connection I354

To get more details on a Nic

	# esxcli network nic get  -n vmnic0

	Advertised Auto Negotiation: true
	Advertised Link Modes: 10baseT/Half, 10baseT/Full, 100baseT/Half, 100baseT/Ful	1000baseT/Full
	Auto Negotiation: true
	Cable Type: Twisted Pair
	Current Message Level: 7
	Driver Info:
	      Bus Info: 0000:00:14.0
	      Driver: igb
	      Firmware Version: 0.0.0
	      Version: 5.2.5
	Link Detected: true
	Link Status: Up by explicit linkSet
	Name: vmnic0
	PHYAddress: 0
	Pause Autonegotiate: true
	Pause RX: false
	Pause TX: false
	Supported Ports: TP
	Supports Auto Negotiation: true
	Supports Pause: true
	Supports Wakeon: true
	Transceiver: internal
	Wakeon: MagicPacket(tm)

To check TCP segmentation offload (TSO) and checksum offload (CSO) settings for a Nic

	# esxcli network nic tso get -n vmnic0
	# esxcli network nic cso get -n vmnic0

If you see some strange behavior while using LACP, you can use the following commands to leave only one interface up to verify if LACP negotiation is reponsible for your issues:

	# esxcli network nic down -n vmnic1
	# esxcli network nic down -n vmnic2
	# esxcli network nic down -n vmnic3

To revert it:

	# esxcli network nic up -n vmnic1
	# esxcli network nic up -n vmnic2
	# esxcli network nic up -n vmnic3

#### VMs

Get a list of all VMs on the compute node

	# esxcfg-vswitch -l
	Switch Name      Num Ports   Used Ports  Configured Ports  MTU     Uplinks
	vSwitch0         1536        1           128               1500
	
	  PortGroup Name        VLAN ID  Used Ports  Uplinks
	  VM Network            0        0
	
	DVS Name         Num Ports   Used Ports  Configured Ports  MTU     Uplinks
	Mgmt_Edge_VDS    1536        13          512               1600    vmnic1,vmnic0
	
	  DVPort ID           In Use      Client
	  897                 1           vmnic0
	  767                 1           vmk2
	  639                 1           vmk1
	  510                 1           vmk0
	  895                 0
	  905                 1           vmk3
	  907                 1           vmnic1
	  506                 1           NSX_Controller_c6aea614-0dc7-40fd-b646-0230608d4709.eth0
	  497                 1           dr-4-bridging-0.eth0
	  127                 1           br-sv-01a.eth0

#### VMkernel Port

	# esxcfg-vmknic -l
	Interface  Port Group/DVPort   IP Family IP Address                              Netmask         	Broadcast       MAC Address       MTU     TSO MSS   Enabled Type
	vmk0       510                 IPv4      192.168.110.52                          255.255.255.0   192.168.	110.255 00:50:56:09:08:3c 1500    65535     true    STATIC
	vmk1       639                 IPv4      10.10.20.52                             255.255.255.0   10.10.20	.255    00:50:56:64:f0:9b 1500    65535     true    STATIC
	vmk2       767                 IPv4      10.10.30.52                             255.255.255.0   10.10.30	.255    00:50:56:65:67:8e 1500    65535     true    STATIC
	vmk3       905                 IPv4      192.168.150.52                          255.255.255.0   192.168.	150.255 00:50:56:6e:5e:e3 1600    65535     true    STATIC

#### ARP Table

	# esxcli network ip neighbor list
	Neighbor         Mac Address        Vmknic    Expiry  State  Type
	---------------  -----------------  ------  --------  -----  -------
	192.168.110.202  00:50:56:8e:52:25  vmk0     674 sec         Unknown
	192.168.110.42   00:50:56:09:45:60  vmk0    1196 sec         Unknown
	192.168.110.10   00:50:56:03:00:2a  vmk0    1199 sec         Unknown
	192.168.110.203  00:50:56:8e:7a:a4  vmk0     457 sec         Unknown
	192.168.110.201  00:50:56:8e:ea:bd  vmk0     792 sec         Unknown
	192.168.110.22   00:50:56:09:11:07  vmk0    1146 sec         Unknown
	10.10.20.60      00:50:56:27:49:6b  vmk1     506 sec         Unknown

#### VXLAN

Dump VXLAN configuration

	# esxcli network vswitch dvs vmware vxlan list
	VDS ID                                           VDS Name        MTU  Segment ID     Gateway IP     Gateway MAC        Network Count  Vmknic Count
	-----------------------------------------------  -------------  ----  -------------  -------------  -----------------  -------------  ------------
	1c ec 0e 50 02 9c a9 21-b6 d8 d0 fc 73 e5 79 69  Mgmt_Edge_VDS  1600  192.168.150.0  192.168.150.2  00:50:56:27:48:7d              2             1

List VXLAN networks, great to see who's the master controller for each VXLAN.

	# esxcli network vswitch dvs vmware vxlan network list --vds-name=<VDS_NAME>
	VXLAN ID  Multicast IP               Control Plane                        Controller Connection  Port Count  MAC Entry Count  ARP Entry Count  MTEP Count
	--------  -------------------------  -----------------------------------  ---------------------  ----------  ---------------  ---------------  ----------
	    5000  N/A (headend replication)  Enabled (multicast proxy,ARP proxy)  192.168.110.202 (up)	          1                1                0           0
	    5004  N/A (headend replication)  Enabled (multicast proxy,ARP proxy)  192.168.110.203 (up)            1                0                0           0	
Get more details on a specific VXLAN wire

	# esxcli network vswitch dvs vmware vxlan network mac list --vds-name=Mgmt_Edge_VDS --vxlan-id=<VXLAN ID>
	Inner MAC          Outer MAC          Outer IP        Flags
	-----------------  -----------------  --------------  --------
	00:50:56:ae:9b:be  ff:ff:ff:ff:ff:ff  192.168.250.51  00001111

But you can also get specific information on the VXLAN like mac, arp, port, mtep and stats like this

For example to list all the remote Mac addresses, pushed by the controller for a specific VNI:

	# esxcli network vswitch dvs vmware vxlan network mac list –-vds-name=<VDS_NAME> --vxlan-id=<VXLAN_ID>

You can get also the remote IP Addresses/Mac Addresses that are still in the local ESXi cache. They timeout after 5' if no traffic.

	# esxcli network vswitch dvs vmware vxlan network arp list --vds-name=<VDS_NAME> --vxlan-id=<VXLAN_ID>

To get a list of remote known ESXi for a VNI. You'll also see who's the MTEP on each Transport Network subnet.

	# esxcli network vswitch dvs vmware vxlan network vtep list --vds-name=<VDS_NAME> --vxlan-id=<VXLAN_ID>

Note: if you don't get the `vtep` argument but he `mtep` one intead, just run `/etc/init.d/hostd restart`

	# esxcli network vswitch dvs vmware vxlan network port list --vds-name=<VDS_NAME> --vxlan-id=<VXLAN_ID>

	# esxcli network vswitch dvs vmware vxlan network stats list --vds-name=<VDS_NAME> --vxlan-id=<VXLAN_ID>	

#### Controller connectivity

To check Controller connectivity from ESXi (VDL= Virtual Distributed Layer 2)

	# net-vdl2 -l
	VXLAN Global States:
	        Control plane Out-Of-Sync:      No
	        UDP port:       8472
	VXLAN VDS:      Mgmt_Edge_VDS
	        VDS ID: 1c ec 0e 50 02 9c a9 21-b6 d8 d0 fc 73 e5 79 69
	        MTU:    1600
	        Segment ID:     192.168.150.0
	        Gateway IP:     192.168.150.2
	        Gateway MAC:    00:50:56:27:48:7d
	        Vmknic count:   1
	                VXLAN vmknic:   vmk3
	                        VDS port ID:    905
	                        Switch port ID: 50331656
	                        Endpoint ID:    0
	                        VLAN ID:        0
	                        IP:             192.168.150.52
	                        Netmask:        255.255.255.0
	                        Segment ID:     192.168.150.0
	                        IP acquire timeout:     0
	                        Multicast group count:  0
	        Network count:  2
	                VXLAN network:  5000
	                        Multicast IP:   N/A (headend replication)
	                        Control plane:  Enabled (multicast proxy,ARP proxy)
	                        Controller:     192.168.110.202 (up)
	                        MAC entry count:        1
	                        ARP entry count:        0
	                        Port count:     1
	                VXLAN network:  5004
	                        Multicast IP:   N/A (headend replication)
	                        Control plane:  Enabled (multicast proxy,ARP proxy)
	                        Controller:     192.168.110.203 (up)
	                        MAC entry count:        0
	                        ARP entry count:        0
	                        Port count:     1

Or

	# esxcli network vswitch dvs vmware vxlan network list -–vds-name <vds name>
	VXLAN ID  Multicast IP               Control Plane                        Controller Connection  Port Count  MAC Entry Count  ARP Entry Count  MTEP Count
	--------  -------------------------  -----------------------------------  ---------------------  ----------  ---------------  ---------------  ----------
	    5000  N/A (headend replication)  Enabled (multicast proxy,ARP proxy)  192.168.110.202 (up)	          1                1                0           0
	    5004  N/A (headend replication)  Enabled (multicast proxy,ARP proxy)  192.168.110.203 (up)            1                0                0           0


If you see a controller down message above, you can fix it by restarting `netcpa` like this

	# /etc/init.d/netcpad restart

Note: netcpa is a user world agent that communicate thru SSL with the NSX Controller.

To check ESXi controller connections.

	# esxcli network ip connection list| grep tcp | grep 1234
	
	tcp         0       0  192.168.110.52:43925  192.168.110.203:1234  ESTABLISHED     44923  newreno  netcpa-worker
	tcp         0       0  192.168.110.52:46026  192.168.110.202:1234  ESTABLISHED     46232  newreno  netcpa-worker
	tcp         0       0  192.168.110.52:39244  192.168.110.201:1234  ESTABLISHED     44923  newreno  netcpa-worker

As you can see, your compute node is connected to all Controllers.

#### Logical Router

First get a list of distributed router (VDR) instances
	
	# net-vdr --instance -l
	
	VDR Instance Information :
	---------------------------
	
	VDR Instance:               default+edge-1:1460487509
	Vdr Name:                   default+edge-1
	Vdr Id:                     1460487509
	Number of Lifs:             3
	Number of Routes:           1
	State:                      Enabled
	Controller IP:              192.168.110.201
	Control Plane Active:       Yes
	Control Plane IP:           192.168.110.52
	Edge Active:                Yes


Dump all the logical interfaces (LIFs) for a VDR instance
	
	# net-vdr --lif -l default+edge-1

	VDR default+edge-1:1460487509 LIF Information :

	Name:                570d45550000000c
	Mode:                Routing, Distributed, Internal
	Id:                  Vxlan:5004
	Ip(Mask):            10.10.10.1(255.255.255.0)
	Connected Dvs:       Mgmt_Edge_VDS
	VXLAN Control Plane: Enabled
	VXLAN Multicast IP:  0.0.0.1
	State:               Enabled
	Flags:               0x2288
	
	Name:                570d45550000000b
	Mode:                Bridging, Sedimented, Internal
	Id:                  Vlan:100
	Bridge Id:           mybridge:1
	Ip(Mask):            0.0.0.0(0.0.0.0)
	Connected Dvs:       Mgmt_Edge_VDS
	Designated Instance: No
	DI IP:               192.168.110.51
	State:               Enabled
	Flags:               0xd4
	
	Name:                570d45550000000a
	Mode:                Bridging, Sedimented, Internal
	Id:                  Vxlan:5000
	Bridge Id:           mybridge:1
	Ip(Mask):            0.0.0.0(0.0.0.0)
	Connected Dvs:       Mgmt_Edge_VDS
	VXLAN Control Plane: Enabled
	VXLAN Multicast IP:  0.0.0.1
	State:               Enabled
	Flags:               0x23d4

Check routing status

	# net-vdr -R -l default+edge-1

	VDR default+edge-1:1460487509 Route Table
	Legend: [U: Up], [G: Gateway], [C: Connected], [I: Interface]
	Legend: [H: Host], [F: Soft Flush] [!: Reject]
	
	Destination      GenMask          Gateway          Flags    Ref Origin   UpTime     Interface
	-----------      -------          -------          -----    --- ------   ------     ---------
	10.10.10.0       255.255.255.0    0.0.0.0          UCI      1   MANUAL   410777     570d45550000000c

ARP information
	
	# net-vdr --nbr -l default+edge-1

	VDR default+edge-1:1460487509 ARP Information :
	Legend: [S: Static], [V: Valid], [P: Proxy], [I: Interface]
	Legend: [N: Nascent], [L: Local], [D: Deleted]
	
	Network           Mac                  Flags      Expiry     SrcPort    Interface Refcnt
	-------           ---                  -----      -------    ---------  --------- ------
	10.10.10.1        02:50:56:56:44:52    VI         permanent  0          570d45550000000c 1


Designated instance statistics
	
	# net-vdr --di —stats

	VDR Designated Instance Statistics:

        RX Pkts:                        0
        RX Bytes:                       0
        TX Pkts:                        0
        TX Bytes:                       0
        ARP Requests:                   0
        ARP Response:                   0
        ARP Resolved:                   0
        Err RX:                         0
        Err TX:                         0
        Err Message too Small:          0
        Err Message too Big:            0
        Err Invalid Version:            0
        Err Invalid Message Type:       0
        Err Instance Not Found:         0
        Err LIF not DI:                 0
        Err No memory:                  0
        Err ARP not found:              0
        Err Proxy not found:            0
        Err DI Remote:                  0

#### Bridging

Dump bridge info
	
	# net-vdr --bridge -l <vdrName>

	VDR default+edge-1:1460487509 Bridge Information :

	Bridge config:
	Name:id             mybridge:1
	Portset name:
	DVS name:           Mgmt_Edge_VDS
	Ref count:          2
	Number of networks: 2
	Number of uplinks:  0
	
	        Network 'vlan-100-type-bridging' config:
	        Ref count:          2
	        Network type:       1
	        VLAN ID:            100
	        VXLAN ID:           0
	        Ageing time:        300
	        Fdb entry hold time:1
	        FRP filter enable:  1
	
	                Network port '50331655' config:
	                Ref count:          2
	                Port ID:            0x3000007
	                VLAN ID:            4095
	                IOChains installed: 0
	
	        Network 'vxlan-5000-type-bridging' config:
	        Ref count:          2
	        Network type:       1
	        VLAN ID:            0
	        VXLAN ID:           5000
	        Ageing time:        300
	        Fdb entry hold time:1
	        FRP filter enable:  1
	
	                Network port '50331655' config:
	                Ref count:          2
	                Port ID:            0x3000007
	                VLAN ID:            4095
	                IOChains installed: 0

Lists MAC table, learnt on both VXLAN and VLAN sides 

	# net-vdr -b --mac default+edge-1

	VDR default+edge-1:1460487509 Bridge Information :

	Network 'vlan-100-type-bridging' MAC address table:
	MAC table on PortID:              0x0
	MAC table paging mode:            0
	Single MAC address enable:        0
	Single MAC address:               00:00:00:00:00:00
	MAC table last entry shown:       00:50:56:91:5e:93 VLAN-VXLAN: 100-0 Port: 50331661
	total number of MAC addresses:    1
	number of MAC addresses returned: 1
	MAC addresses:
	Destination Address  Address Type  VLAN ID  VXLAN ID  Destination Port  Age
	-------------------  ------------  -------  --------  ----------------  ---
	00:50:56:91:5e:93    Dynamic           100         0          50331661  0
	
	
	Network 'vxlan-5000-type-bridging' MAC address table:
	MAC table on PortID:              0x0
	MAC table paging mode:            0
	Single MAC address enable:        0
	Single MAC address:               00:00:00:00:00:00
	MAC table 	last entry shown:       00:50:56:ae:9b:be VLAN-VXLAN: 0-5000 Port: 50331650
	total number of MAC addresses:    1
	number of MAC addresses returned: 1
	MAC addresses:
	Destination Address  Address Type  VLAN ID  VXLAN ID  Destination Port  Age
	-------------------  ------------  -------  --------  ----------------  ---
	00:50:56:ae:9b:be    Dynamic             0      5000          50331650  0

Dump statistics (output not shown)

	# net-vdr -b --stats default+edge-1

#### Distributed Firewall 

To investigate the Distributed Firewall Rules applied to a Virtual Machine vNic, first get the VM UUID (vcUuid) for it using (output abridged)

	# summarize-dvfilter
	..
	world 230869 vmm0:br-sv-01a vcUuid:'50 11 97 6c fa 73 a7 5b-a7 0e 36 1f a2 f5 84 38'
	..

Now find the filter name for that VM UUID

	# vsipioctl getfilters

	Filter Name              : nic-230869-eth0-vmware-sfw.2
	VM UUID                  : 50 11 97 6c fa 73 a7 5b-a7 0e 36 1f a2 f5 84 38
	VNIC Index               : 0
	Service Profile          : --NOT SET--

Use the filter name above to list the associated Distributed Firewall Rules

	# vsipioctl getrules -f nic-230869-eth0-vmware-sfw.2

To details Addresses Sets

 	# vsipioctl getaddrsets -f nic-230869-eth0-vmware-sfw.2

#### Packet Capture

vSphere 5 offers a new command, `pktcap-uw` to capture packet at different level of the processing.

![][nsxv-pktcap]

You can get look at all possibilities
	
	pktcap-uw -h |more

As you can see on the diagram above, we can now capture traffic at the `vmnic`, `vmknic`, 'vnic' level. 

Let see how it works from the outside world to the VM. I'm not going to include the ouput of the command here, I advice you to try on your hosts instead. By the way I also advice to save the output to a file in pcap format with `-o ./save.pcap`, you'll then be able to open it from Wireshark.

####  uplink/vmnic

You can open up the DVUplinks section of your VDS to get the name of your uplink interface. Here we'll be using `vmnic0`. So to capture packets received on this uplink, use

	pktcap-uw --uplink vmnic0

By default it will only capture received traffic (RX), to capture packets sent on the uplink to the outside world use the `--capture` argument like this

	pktcap-uw --uplink vmnic0 --capture UplinkSnd

We'll details all the filtering options at the end of this section, but in the meantime you can for example filter out only **ICMP** packet received on a specific destination by using `--proto 0x01` and `--destip <ip>`

	pktcap-uw –uplink vmnic0 –proto 0x01 –dstip <IP>

Or to capture ICMP Packets that are sent on vmnic0 from an IP Address 192.168.25.113

	pktcap-uw --uplink vmnic0 --capture UplinkSnd –proto 0x01 --srcip 192.168.25.113

You can also capture **ARP** packets

	pktcap-uw --uplink vmnic0 –ethtype 0x0806 

#### vmknic - Virtual Adapters

Capture packets reaching vmknic adapter is also possible, just use `--vmk` argument.

	pktcap-uw --vmk vmk0 

#### Switchport

To capture on a specific switchport, you first have to get the ID of the port. Launch

	# esxtop

Type `n`, to get a list of all the ports with the corresponding attachment. Take note of Port ID of the port you're interested in and use a `--switchport` argument like this

	pktcap-uw –switchport <port-id> –-proto 0x01 

#### Traffic direction

For `–switchport`, `–vmk`, `–uplink`, `–dvfilter`, direction of traffic is specified using `--dir 0` for inbound and `--dir 1` for outbound but inbound is assumed.

	0- Rx (Default)
	1- Tx

So don't be surprised, pktcap-uw doesn't work like tcpdump and by default only capture the received (RX) traffic. Don't forget to change that if necessary by specifying `--dir 1`, it will switch the capture to the Transmit (Tx) direction.

#### Argument and Filtering Options

`-o save.pcap` to save capture to a file in pcap format  
`-c 25` capture only 25 packets  
`–-vxlan <segment id>` to specify VXLAN VNI of flow  
`--vlan <VLANID>` filter for VLAN ID  
`–-ip <x.x.x.x>` filter for SRC or DST   
`--srcmac <xx:xx:xx:xx:xx>` filter for source mac address  
`--dstmac <xx:xx:xx:xx:xx>` filter for source mac address  
`--srcip <x.x.x.x[/<range>]>` filter for source IP  
`--dstip <x.x.x.x[/<range>]>` filter for destination IP  
`--dstport <DSTPORT>` to specify a TCP destination Port  
`--srcport <SRCPORT>` to specify a TCP source Port  
`--tcpport <PORT>` filter for source or destination Port  
`--proto 0x<IPPROTYPE>` filter on hexadecimal protocol id: 0x01 for ICMP, 0x06 for TCP, 0x11 for UDP.list [here](http://en.wikipedia.org/wiki/List_of_IP_protocol_numbers)  
`--ethtype 0x<ETHTYPE>` filter on ethernet type, 0x0806 for ARP  

#### Decoding capture

We've shown you how to save the captured packets to a file, to get a quick overview of the kind of traffic passing by, you can decode the pcap using tcpdump like this

	tcpdump-uw -r save.pcap

But using Wireshark will give you a better vision of the traffic, with all the details.

#### Tracing

If you are interested in seeing even more details on the processing of the packet through the ESXi TCP/IP stack, just add `--trace` argument to see packet traversing the ESXi network stack. Looks for Drop message that indicate something went wrong in the processing.

#### Drops

When things don't work as you expect, one really usefull command is

	pktcap-uw --capture Drop 

You should see here some errors like `VLAN Mismatch` or something else that will give you a hint about why traffic isn't flowing as you would expect.

#### DVFilter

This command captures packets as seen by the dvfilter (before the filtering happens)
	
	pktcap-uw --capture PreDVFilter --dvfilterName <filter name>

This command captures packets after being subject to the dvfilter.

	pktcap-uw --capture PostDVFilter --dvfilterName <filter name>   

#### Capture point

You can get a list of all possible capture point with `-A`

	pktcap-uw -A
	Supported capture points:
        1: Dynamic -- The dynamic inserted runtime capture point.
        2: UplinkRcv -- The function that receives packets from uplink dev
        3: UplinkSnd -- Function to Tx packets on uplink
        4: Vmxnet3Tx -- Function in vnic backend to Tx packets from guest
        5: Vmxnet3Rx -- Function in vnic backend to Rx packets to guest
        6: PortInput -- Port_Input function of any given port
        7: IOChain -- The virtual switch port iochain capture point.
        8: EtherswitchDispath -- Function that receives packets for switch
        9: EtherswitchOutput -- Function that sends out packets, from switch
        10: PortOutput -- Port_Output function of any given port
        11: TcpipDispatch -- Tcpip Dispatch function
        12: PreDVFilter -- The DVFIlter capture point
        13: PostDVFilter -- The DVFilter capture point
        14: Drop -- Dropped Packets capture point
        15: VdrRxLeaf -- The Leaf Rx IOChain for VDR
        16: VdrTxLeaf -- The Leaf Tx IOChain for VDR
        17: VdrRxTerminal -- Terminal Rx IOChain for VDR
        18: VdrTxTerminal -- Terminal Tx IOChain for VDR
        19: PktFree -- Packets freeing point


Let me share with you a little bit more details about some of them.

`PortOutput` show traffic delivered from the vSwitch to the Guest when used with switch port or to the physical adapter if used with a physical adapter  
`VdrRxLeaf` - Capture packets at the receive leaf I/O chain of a dynamic router in VMware NSX. Use this capture point together with the --lifID option  
`VdrRxTerminal` - Capture packets at the receive terminal I/O chain of a dynamic router in VMware NSX. Use this capture point together with the --lifID option  
`VdrTxLeaf` - Capture packets at the transmit leaf I/O chain of a dynamic router in VMware NSX. Use this capture point together with the --lifID option  
`VdrTxTerminal` - Capture packets at the transmit terminal I/O chain of a dynamic router in VMware NSX. Use this capture point together with the --lifID option  
`

#### CTRL-D

Never press `CTRL-D` to interupt a running packet capture or you'll be left with a background process still running. If you've done it you can kill it like this

	kill $(lsof |grep pktcap-uw |awk '{print $1}'| sort -u)

Then check it was killed
	
	lsof |grep pktcap-uw |awk '{print $1}'| sort -u  

### NSX Edge CLI

NSX Edge offers lots of Layer 4 to Layer 7 services, to name a few :

* VPN
* SSL-VPN
* LB
* FW
* DHCP Relay

Lets now details the command line interface available from a SSH connection to an NSX Edge

	show ip route
	show ip ospf neighbor
	show ip ospf database
	
	show configuration {ospf|bgp|isis|static-routing}
	show configuration {firewall|nat|dhcp|dns}
	show configuration {loadbalancer|ipec|sslvpn-plus} 
	show interface [IFNAME]
	show firewall
	show ip {route|ospf|bgp|forwarding}
	show arp
	show system {cpu|memory|network-stats|storage|uptime}
	show service {dhcp|dns|highavailability|ipsec|loadbalancer|sslvpn-plus}
	show log {follow|reverse}
	show floatable

But one of the most convenient one is the following which enable you to dump all ingress/egress traffic on a specific edge interface

	debug packet display interface vNic_0

You'll then get a live display of what's flowing on that virtual network interface:

	tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
	listening on vNic_0, link-type EN10MB (Ethernet), capture size 65535 bytes
	22:43:44.190770 IP 172.16.16.3 > 224.0.0.5: OSPFv2, Hello, length 48
	22:43:45.914868 IP 172.16.16.1.17773 > 192.168.2.20.8080: Flags [S], seq 1790616868, win 14600, options [mss 1460,nop,nop,sackOK,nop,wscale 3], length 0
	22:43:45.915713 IP 192.168.2.20.8080 > 172.16.16.1.17773: Flags [S.], seq 841084052, ack 1790616869, win 29200, options [mss 1460,nop,nop,sackOK,nop,wscale 6], length 0
	22:43:45.915750 IP 172.16.16.1.17773 > 192.168.2.20.8080: Flags [.], ack 1, win 1825, length 0
	22:43:45.915990 IP 172.16.16.1.17773 > 192.168.2.20.8080: Flags [P.], seq 1:101, ack 1, win 1825, length 100
	22:43:45.916497 IP 192.168.2.20.8080 > 172.16.16.1.17773: Flags [.], ack 101, win 457, length 0
	22:43:45.922654 IP 192.168.2.20.8080 > 172.16.16.1.17773: Flags [P.], seq 4381:4434, ack 101, win 457, length 53
	22:43:45.922674 IP 192.168.2.20.8080 > 172.16.16.1.17773: Flags [F.], seq 4434, ack 101, win 457, length 0
	22:43:45.922842 IP 172.16.16.1.17773 > 192.168.2.20.8080: Flags [.], ack 1, win 1825, options [nop,nop,sack 1 {4381:4434}], length 0

You can use the same filtering syntax as the one used by [tcpdump](http://www.wains.be/pub/networking/tcpdump_advanced_filters.txt), for example :

	debug packet display interface vNic_0 icmp

If you have multiple words for your filter, just add underscore between them

	debug packet display interface vNic_0 dst_port_443

### Logs

#### Controller Logs

Check ESXi connectivity issues from the Controller

	show log cloudnet/cloudnet_java-vnet-controller.<start-time-stamp>.log

#### NSX Manager

SSH (l: admin) to the NSX Manager and use the following command to access logs

	nsxmgr-l-01a> show manager log follow

You can switch over to a unix shell using

	nsxmgr-l-01a> enable
	Password: <your NSX-Mgr pwd>
	nsx_manager# st e
	Password: <ASK NSX SUPPORT FOR PASSWORD>
	[root@nsxmgr-l-01a ~]#

#### ESXi Logs

`/var/log/esxupdate.log` check this file if you have VIB installation issues  
`/var/log/vmkernel.log` Distributed Firewall logs are sent to this file  
`/var/log/netcpa.log` User World Agent logs  

#### EAM Logs

EAM logs should be checked when installation of the VIBs module fails.

`/storage/log/vmware/vpx/eam.log` on Linux vCenter  
`ProgramData/VMware/VMware VirtualCenter/Logs/` on Windows vCenter

### Advanced troubleshooting tips & tricks

If you want to troubleshoot your User World Agent, you can increase the netcpa log level like this:

Start by stopping the daemon

	# /etc/init.d/netcpad stop

Enable write permisions on netcpa's config file:

	# chmod +wt /etc/vmware/netcpa/netcpa.xml

Increase log level:
	
	# vi /etc/vmware/netcpa/netcpa.xml

Change the XML's /config/log/level value to "verbose", save and restart netcpad

	# /etc/init.d/netcpad start

### Conclusion

I hope you'll accelerate your troubleshooting session by having a great understanding of all the internals. 

[nsxv-controller-status]: /images/posts/nsxv-controller-status.png
[nsxv-clusters-status]: /images/posts/nsxv-clusters-status.png
[nsxv-preparation-status]: /images/posts/nsxv-preparation-status.png
[nsxv-controller-vm]: /images/posts/nsxv-controller-vm.png
[nsxv-controller-clistatus]: /images/posts/nsxv-controller-clistatus.png
[nsxv-vds]: /images/posts/nsxv-vds.png
[nsxv-pktcap]: /images/posts/nsxv-pktcap.png
