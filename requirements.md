# Technical requirements

## Network

The T.rex exhibition consists of (max.) four exhibits with one computer each + one on/off controller unit. The controller unit can be used to power on and off the exhibits and allows remote management of the exhibits by Naturalis. 
As outlined in [the network design](img/networkdesign.png) only the controller unit needs will be directly connected to the LAN. The requirements for the network access are:

* A DHCP server that gives out an (reserved) IP address to the controller unit and a DNS-server

* Access from the controller unit to the internet:
  *  allow outbound traffic to any udp/tcp port 5938 (teamviewer)
  
  * As an absolute minimum access to the Naturalis VPN server  t-rex-in-town.naturalis.io  port 51820 UDP (current ipaddess is 145.136.241.119)

  * In order for the graffitti exhibit to send out mails outgoing traffic to smtp.gmail.com (port 25) and smtp-relay.gmail.com (port 587) should be allowed.

  * In order for the computers to run updates outgoing traffic over HTTP (port 80) and HTTPS (port 443) should be allowed.

* Access from the internet to the controller:

  * In case of issues with the controller (i.e. the VPN connection) we need to be able to reach the controller unit over SSH. We'd like to receive a public IPv4 address (possibly NAT) and relevant port number by which we can reach the controller.

## Power

As explained in detail in the [user manual](usermanual.md) the controller unit can power on and off the exhibits. This is necessary for the exhibit computers to shutdown gracefully in order to prevent system errors. Depending on the situation on site the controller unit might be powered off as well. The configuration of the unit will be changed according to the specific situation on site.

