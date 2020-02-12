Setup cross geo high available connection between on premise and Azure with IPSec VPN 
====================
Customer have Azure workload at Azure China North and Azure Global East Asia. <br>
They use IPSec VPN connect from on premise to Azure China North and extend IPSec VPN to East Asia. <br>
They deploy BGP on top of IPSec VPN to exchange customer route. <br>
Customer wants to setup another IPSec VPN to Azure China East and provide cross geo high available connection. <br>
Either link level failure or device level failure will not impact customer connectivity from on premise to Azure. <br>

Topology
-------------
![](https://github.com/yinghli/azure-vpn-csr1000v/blob/master/Topology.png)

Azure VPN Setup 
-----------------
In Azure side, we will use Azure Portal to setup all vpn configuration. PowerShell and Azure CLI can do the same setup. <br>
We will use below parameters to setup. <br>


Parameters            | On Premise     |China North   |China East    |East Asia
----------------------| -------------  |-----------   |---------     |------------
Azure BGP ASN         |65000           |65001         |65002         |65003
Azure BGP Public IP   |123.121.212.64  |40.125.213.180|139.219.190.98|23.97.67.99
Azure BGP peer IP     |10.255.1.1      |10.2.1.254    |10.2.3.254    |10.6.0.254
Azure BGP peer IP     |10.255.1.2      |              |              |
Local Network         |172.16.1.0/24   |10.2.0.0/23   |10.2.3.0/24   |10.6.0.0/16
Local Network         |                |10.2.10.0/23  |10.2.6.0/23   |

For more information about how to setup IPsec VPN with Azure VPN gateway, plesse check [here](https://github.com/yinghli/azure-vpn-asa/blob/master/README.md)

On Premise Cisco CSR1000v VPN Setup 
-----------------

## Setup IKEv2
Setup 2 IKEv2 profile, one is to Azure China North(BJ), the other is to Azure China East(SH). <br>
```
crypto ikev2 proposal azure-proposal
 encryption aes-cbc-256 aes-cbc-128 3des
 integrity sha1
 group 2
!
crypto ikev2 policy azure-policy
 proposal azure-proposal
!
crypto ikev2 keyring BJ
 peer 40.125.213.180
  address 40.125.213.180
  pre-shared-key cisco123
 !
!
crypto ikev2 keyring SH
 peer 139.219.190.98
  address 139.219.190.98
  pre-shared-key cisco123
 !
!
crypto ikev2 profile BJ
 match identity remote address 40.125.213.180 255.255.255.255
 authentication local pre-share
 authentication remote pre-share
 keyring local BJ
!
crypto ikev2 profile SH
 match identity remote address 139.219.190.98 255.255.255.255
 authentication local pre-share
 authentication remote pre-share
 keyring local SH
```
## Setup IPSec
Setup 2 IPSec profile, one is to Azure China North(BJ), the other is to Azure China East(SH). <br>
```
crypto ipsec transform-set azure-ipsec-proposal-set esp-aes 256 esp-sha-hmac
 mode tunnel
!
crypto ipsec profile BJ
 set transform-set azure-ipsec-proposal-set
 set ikev2-profile BJ
!
crypto ipsec profile SH
 set transform-set azure-ipsec-proposal-set
 set ikev2-profile SH

```
## Setup Tunnel Interface
Setup 2 tunnel interface, use 2 seperate IP address, one is to Azure China North(BJ), the other is to Azure China East(SH). <br>
```
interface Loopback0
 ip address 10.255.1.1 255.255.255.255
!
interface Loopback1
 ip address 10.255.1.2 255.255.255.255
!
interface Tunnel1
 description to BJ
 ip unnumbered Loopback0
 ip tcp adjust-mss 1350
 tunnel source GigabitEthernet4
 tunnel mode ipsec ipv4
 tunnel destination 40.125.213.180
 tunnel protection ipsec profile BJ
!
interface Tunnel2
 description SH
 ip unnumbered Loopback1
 ip tcp adjust-mss 1350
 tunnel source GigabitEthernet4
 tunnel mode ipsec ipv4
 tunnel destination 139.219.190.98
 tunnel protection ipsec profile SH
!
```
## Setup static route
Setup 2 static route to remote BGP peer IP address.<br>
```
ip route 10.2.1.254 255.255.255.255 Tunnel1
ip route 10.2.3.254 255.255.255.255 Tunnel2
```
## Setup BGP
Setup route-map to advertise local route only.
```
ip prefix-list bgp seq 5 permit 172.16.1.0/24
!
route-map bgp permit 10
 match ip address prefix-list bgp
!
```
Setup normal BGP neighbors information
```
router bgp 65000
 bgp log-neighbor-changes
 bgp bestpath as-path multipath-relax
 neighbor 10.2.1.254 remote-as 65001
 neighbor 10.2.1.254 ebgp-multihop 3
 neighbor 10.2.1.254 update-source Loopback0
 neighbor 10.2.3.254 remote-as 65002
 neighbor 10.2.3.254 ebgp-multihop 3
 neighbor 10.2.3.254 update-source Loopback1
 !
 address-family ipv4
  network 172.16.1.0 mask 255.255.255.0
  neighbor 10.2.1.254 activate
  neighbor 10.2.1.254 route-map bgp out
  neighbor 10.2.3.254 activate
  neighbor 10.2.3.254 route-map bgp out
  maximum-paths 16
 exit-address-family
!
```
Verify IPSec VPN and BGP information
-----------------
## IKEv2 Status
IKEv2 have 2 SA,  one is to Azure China North(BJ), the other is to Azure China East(SH). <br>
```
CSR1000vOnPrem#show crypto ikev2 sa
 IPv4 Crypto IKEv2  SA

Tunnel-id Local                 Remote                fvrf/ivrf            Status
2         192.168.1.17/4500     139.219.190.98/4500   none/none            READY
      Encr: AES-CBC, keysize: 256, PRF: SHA1, Hash: SHA96, DH Grp:2, Auth sign: PSK, Auth verify: PSK
      Life/Active Time: 86400/13936 sec

Tunnel-id Local                 Remote                fvrf/ivrf            Status
3         192.168.1.17/4500     40.125.213.180/4500   none/none            READY
      Encr: AES-CBC, keysize: 256, PRF: SHA1, Hash: SHA96, DH Grp:2, Auth sign: PSK, Auth verify: PSK
      Life/Active Time: 86400/13911 sec

 IPv6 Crypto IKEv2  SA

```
## IPSec Status
IPSec have 2 SA,  one is to Azure China North(BJ), the other is to Azure China East(SH). <br>
```
CSR1000vOnPrem#show crypto ipsec sa

interface: Tunnel1
    Crypto map tag: Tunnel1-head-0, local addr 192.168.1.17

   protected vrf: (none)
   local  ident (addr/mask/prot/port): (0.0.0.0/0.0.0.0/0/0)
   remote ident (addr/mask/prot/port): (0.0.0.0/0.0.0.0/0/0)
   current_peer 40.125.213.180 port 4500
     PERMIT, flags={origin_is_acl,}
    #pkts encaps: 696, #pkts encrypt: 696, #pkts digest: 696
    #pkts decaps: 1079, #pkts decrypt: 1079, #pkts verify: 1079
    #pkts compressed: 0, #pkts decompressed: 0
    #pkts not compressed: 0, #pkts compr. failed: 0
    #pkts not decompressed: 0, #pkts decompress failed: 0
    #send errors 0, #recv errors 0

     local crypto endpt.: 192.168.1.17, remote crypto endpt.: 40.125.213.180
     plaintext mtu 1422, path mtu 1500, ip mtu 1500, ip mtu idb GigabitEthernet4
     current outbound spi: 0xF58A1506(4119467270)
     PFS (Y/N): N, DH group: none

interface: Tunnel2
    Crypto map tag: Tunnel2-head-0, local addr 192.168.1.17

   protected vrf: (none)
   local  ident (addr/mask/prot/port): (0.0.0.0/0.0.0.0/0/0)
   remote ident (addr/mask/prot/port): (0.0.0.0/0.0.0.0/0/0)
   current_peer 139.219.190.98 port 4500
     PERMIT, flags={origin_is_acl,}
    #pkts encaps: 694, #pkts encrypt: 694, #pkts digest: 694
    #pkts decaps: 1059, #pkts decrypt: 1059, #pkts verify: 1059
    #pkts compressed: 0, #pkts decompressed: 0
    #pkts not compressed: 0, #pkts compr. failed: 0
    #pkts not decompressed: 0, #pkts decompress failed: 0
    #send errors 0, #recv errors 0

     local crypto endpt.: 192.168.1.17, remote crypto endpt.: 139.219.190.98
     plaintext mtu 1422, path mtu 1500, ip mtu 1500, ip mtu idb GigabitEthernet4
     current outbound spi: 0xCE16B110(3457593616)
     PFS (Y/N): N, DH group: none
```
## BGP Status
BGP have 2 neighbors, one is to Azure China North(BJ), the other is to Azure China East(SH). <br>
```
CSR1000vOnPrem#show ip bgp summary
BGP router identifier 10.255.1.2, local AS number 65000
BGP table version is 23, main routing table version 23
11 network entries using 2728 bytes of memory
19 path entries using 2432 bytes of memory
2 multipath network entries and 4 multipath paths
7/4 BGP path/bestpath attribute entries using 1848 bytes of memory
6 BGP AS-PATH entries using 208 bytes of memory
0 BGP route-map cache entries using 0 bytes of memory
0 BGP filter-list cache entries using 0 bytes of memory
BGP using 7216 total bytes of memory
BGP activity 68/57 prefixes, 376/357 paths, scan interval 60 secs

Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.2.1.254      4        65001     140     134       23    0    0 01:54:24        9
10.2.3.254      4        65002     143     139       23    0    0 01:54:24        9
```
BGP route optimaztion
-------------------------
## Inbound Route
On premise have two IPSec tunnel, for East Asia route 10.6.0.0/16, on premise will have same route from China North and China East. <br>
We can setup ```bgp bestpath as-path multipath-relax``` and ```maximum-paths 16``` to support equal cost multi path(ECMP) to do load sharing between those two IPSec tunnel. <br>
Current BGP route table for 10.6.0.0/16, both route is marked as "multipath" and best route. 
```
CSR1000vOnPrem#show ip bgp 10.6.0.0/16
BGP routing table entry for 10.6.0.0/16, version 19
Paths: (2 available, best #2, table default)
Multipath: eBGP
  Not advertised to any peer
  Refresh Epoch 1
  65002 65003
    10.2.3.254 from 10.2.3.254 (10.2.3.254)
      Origin IGP, localpref 100, valid, external, multipath(oldest)
      rx pathid: 0, tx pathid: 0
  Refresh Epoch 1
  65001 65003
    10.2.1.254 from 10.2.1.254 (10.2.1.254)
      Origin IGP, localpref 100, valid, external, multipath, best
      rx pathid: 0, tx pathid: 0x0
```
If we wants to select one of them as primary path and the other as secondary path, we can setup weight or local preference attribute to influence the inbound route selection. BGP inbound route will influence outbound traffic. <br>
If we wants to use Weight, setup ```neighbor 10.2.1.254 weight 500``` to select China North (65001) as the primary path for network 10.6.0.0/16.<br>
From the output, we can see that local route already choose weight 500 route as best. 
```
CSR1000vOnPrem#show ip bgp 10.6.0.0/16
BGP routing table entry for 10.6.0.0/16, version 32
Paths: (2 available, best #2, table default)
Multipath: eBGP
  Not advertised to any peer
  Refresh Epoch 1
  65002 65003
    10.2.3.254 from 10.2.3.254 (10.2.3.254)
      Origin IGP, localpref 100, valid, external
      rx pathid: 0, tx pathid: 0
  Refresh Epoch 1
  65001 65003
    10.2.1.254 from 10.2.1.254 (10.2.1.254)
      Origin IGP, localpref 100, weight 500, valid, external, best
      rx pathid: 0, tx pathid: 0x0
```
If we try to use local preference attribute, we can use route-map to setup 10.6.0.0/16 higher local preference attribute and make it as best. 
```
ip prefix-list lp seq 5 permit 10.6.0.0/16
!
route-map bgplp permit 10
 match ip address prefix-list lp
 set local-preference 200
!
route-map bgplp permit 20
!
```
From the output, we can see that local route already choose local preference 200 route as best. 
```
CSR1000vOnPrem#show ip bgp 10.6.0.0/16
BGP routing table entry for 10.6.0.0/16, version 41
Paths: (2 available, best #2, table default)
Multipath: eBGP
  Not advertised to any peer
  Refresh Epoch 1
  65002 65003
    10.2.3.254 from 10.2.3.254 (10.2.3.254)
      Origin IGP, localpref 100, valid, external
      rx pathid: 0, tx pathid: 0
  Refresh Epoch 1
  65001 65003
    10.2.1.254 from 10.2.1.254 (10.2.1.254)
      Origin IGP, localpref 200, valid, external, best
      rx pathid: 0, tx pathid: 0x0
```
## Outbound Route
For outbound route, typically we will use AS-PATH to influence inbound traffic. Shortest AS-PATH will be best route. <br>
By default, East Asia VPN gateway will have two path to access on premise 172.16.1.0/24. <br>
We can prepend AS-PATH to influence Azure side choosing which path to send traffic. 
```
ip prefix-list bgp seq 5 permit 172.16.1.0/24
!
!
route-map bgppreas permit 10
 match ip address prefix-list bgp
 set as-path prepend 65000
!
```
In BGP neighbor, we will ask East Asia to use China North as primary route, so we prepend AS-PATH to China East BGP neighbor. ```
neighbor 10.2.3.254 route-map bgppreas out```. <br>
If we look at the Azure VPN gateway using powershell ```Get-AzureRmVirtualNetworkGatewayLearnedRoute -VirtualNetworkGatewayName HKGW -ResourceGroupName HKG```, we will see route 172.16.1.0/24 from different BGP neighbor have different AS-PATH attribute. From China North have shortest AS-PATH, East Asia VPN gateway will choose this as best route. 
```
AsPath       : 65001-65000
LocalAddress : 10.6.0.254
Network      : 172.16.1.0/24
NextHop      : 10.2.1.254
Origin       : EBgp
SourcePeer   : 10.2.1.254
Weight       : 32768

AsPath       : 65002-65000-65000
LocalAddress : 10.6.0.254
Network      : 172.16.1.0/24
NextHop      : 10.2.3.254
Origin       : EBgp
SourcePeer   : 10.2.3.254
Weight       : 32768
```
