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

For more information about how to setup IPsec VPN with Azure VPN gateway, plesse check ![here](https://github.com/yinghli/azure-vpn-asa/blob/master/README.md)

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
