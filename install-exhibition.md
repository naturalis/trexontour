# Installation instructions for the T.rex exhibition

## Network

An overview of the network setup [can be found here](img/networkdesign.png).

## On/off unit (a.k.a. Stargazer)

The on/off controller unit has two network interfaces:

* Network interface 1: should be connected to the local network (LAN) of the museum.
* Network interface 2: should be connected to a port on the unmanaged 8-port switch.


## Exhibits

Each exhibit has one network port that should be connected to a port on the unmanaged 8-port switch.

All exhibits are configured with static IPv4 settings:

Property|Setting
--------|-------
IPv4 address|`172.16.61.x`([see the network design](img/networkdesign.png) for an overview)
Subnet mask|`255.255.255.0`
Default gateway|`172.16.61.1`
DNS1|`8.8.8.8`
DNS2|`8.8.4.4`

The exhibit computers are configured to:
 * Start up after power is restored.
 * Start up when a magic Wake on LAN (WoL) packet is received.
