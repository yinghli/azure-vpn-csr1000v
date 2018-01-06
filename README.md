Setup cross geo high available connection between on premise and Azure with IPSec VPN 
====================
Customer have Azure workload at Azure China North and Azure Global East Asia. <br>
They use IPSec VPN connect from on premise to Azure China North and extend IPSec VPN to East Asia. <br>
They deploy BGP on top of IPSec VPN to exchange customer route. <br>
Customer wants to setup another IPSec VPN to Azure China East and provide cross geo high available connection. <br>
Either link level failure or device level failure will not impact customer connectivity from on premise to Azure. <br>

Topology
-------------

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
