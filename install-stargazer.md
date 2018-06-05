# Installation procedure for Stargazer

## Install Ubuntu

1. Download Ubuntu server
1. Make a startup disk:
   `sudo if=ubuntu-16.04.2-server-amd64.iso of=/dev/sdX bs=1M`
1. Boot USB disk
1. Choose primary network interface: enp1s0
1. Hostname: stargazer-001.ontour.naturalis.nl
1. Full name for the new user: Ubuntu
1. Username for your account: ubuntu
1. Password for the new user: zie standaard wachtwoord Keepass
1. Encryption: no
1. Time zone: Europe/Amsterdam
1. Partitioning method: Guided - use entire disk and set up LVM
1. Select disk to partition: sda
1. Amount of volume group to use for guided partitioning: 100%
1. HTTP proxy information: `<blank>`
1. How do you want to manage upgrades on this system: Install security updates automatically
1. Choose software to install:
   * standard system utilities
   * OpenSSH server
1. Install the GRUB boot loader to the master boot record: Yes
1. Device for boot loader installation: /dev/sda

## Configure network

Based on this HOWTO configure the network settings:

1. Install firewalld and dnsmasq:
  `sudo apt install firewalld dnsmasq`
1. Create the network config directory for systemd-networkd:
   `mkdir /etc/systemd/network`
1. Add the configuration for the primary interface `enp1s0.network`:

   ```ini
   [Match]
   Name=enp1s0

   [Network]
   Description=Public interface
   DHCP=yes
   IPForward=yes
   ```

   In case there is no DHCP server available use this configuration (adapt it to
   local circumstances):

   ```ini
   [Match]
   Name=enp1s0

   [Network]
   Description=Public interface
   Address=10.1.10.9/24
   Gateway=10.1.10.1
   DNS=10.1.10.1
   IPForward=yes
   ```

1. Add the configuration for the LAN interface `enp2s0.network`:

   ```ini
   [Match]
   Name=enp2s0

   [Network]
   Description=LAN interface
   Address=172.16.61.1/24
   IPForward=yes
   ```

1. Disable other networking services and enable systemd-networkd:

   ```bash
   systemctl disable network
   systemctl disable NetworkManager
   systemctl enable systemd-networkd
   ```

1. Also, let’s be sure to use systemd-resolved to handle our /etc/resolv.conf:

   ```bash
   systemctl enable systemd-resolved
   systemctl start systemd-resolved
   rm -f /etc/resolv.conf
   ln -s /run/systemd/resolve/resolv.conf /etc/resolv.conf
   ```

1. Enable and start dnsmasq:

   ```bash
   systemctl enable dnsmasq
   systemctl start dnsmasq
   ```

1. Enable masquerading on both interfaces, trust all internal traffic and allow
   access to web interface:

   ```bash
   firewall-cmd --zone=public --add-interface=enp1s0
   firewall-cmd --zone=public --add-masquerade
   firewall-cmd --zone=trusted --add-interface=enp2s0
   firewall-cmd --zone=trusted --add-masquerade
   firewall-cmd --zone=public --remove-service=dns
   firewall-cmd --zone=public --add-port=8080/tcp
   firewall-cmd --zone=public --add-port=5000/tcp
   firewall-cmd --runtime-to-permanent
   ```

1. Enable firewalld:

   `systemctl enable firewalld`

## Install docker

1. Follow the [instructions for installation of Docker CE on
   Ubunu](https://store.docker.com/editions/community/docker-ce-server-ubuntu)
1. Configure Docker to start on boot:
   `sudo systemctl enable docker`

## Install docker containers

1. Clone the stargazer repo:

   ```bash
   git clone https://github.com/MakeExpose/stargazer.git
   cd stargazer
   ```

1. Copy and edit the environment file:

   ```bash
   cp .env.example .env
   vim .env
   ```

1. Move the VPN configuration, key and secret to the configured directory.

1. Copy all the systemd unit files to the right location:

   ```bash
   sudo cp *.(timer|service) /etc/systemd/system/
   systemctl enable \
   stargazer-api.service \
   stargazer-web.service \
   site-to-site-vpn.service \
   site-to-site-vpn.timer
   sudo systemctl daemon-reload
   ```

1. Start the docker containers:

   ```bash
   systemctl start \
   stargazer-api.service \
   stargazer-web.service \
   site-to-site-vpn.service \
   site-to-site-vpn.timer
   ```

1. Optional: enable the stargazer-autopower service to poweron the exhibition
   directly after boot:

   ```bash
   systemctl enable stargazer-autopower.service
   ```

## Install kiosk browser

1. Install packages:
   `sudo apt -y install xorg openbox build-essential`
1. Add kiosk user:

   `adduser kiosk`

1. Add this content to `/home/kiosk/.profile` to start X:

   ```bash
   # Startx Automatically
   . startx -- -nocursor
   logout
   ```

1. Autologin kiosk user:

   `mkdir /etc/systemd/system/getty@tty1.service.d`

   And add this to `/etc/systemd/system/getty@tty1.service.d/override.conf`:

   ```bash
   [Service]
   ExecStart=
   ExecStart=-/sbin/agetty -a kiosk --noclear %I $TERM
   ```

1. Install Chromium
   `sudo apt -y install chromium-browser`

1. Add autostart config file:

   ```bash
   sudo su kiosk
   mkdir /home/kiosk/.config/openbox
   vim /home/kiosk/.config/openbox/autostart.sh
   chmod 644 /home/kiosk/.config/openbox/autostart.sh
   ```

   And add these lines:

   ```bash
   sed -i 's/"exited_cleanly":false/"exited_cleanly":true/' ~/.config/chromium/'Local State'
   sed -i 's/"exited_cleanly":false/"exited_cleanly":true/; s/"exit_type":"[^"]\+"/"exit_type":"Normal"/' ~/.config/chromium/Default/Preferences
   /bin/sleep 90
   chromium-browser --disable-infobars --disable-session-crashed-bubble \
   --kiosk --enable-kiosk-mode --enabled --touch-events --touch-events-ui \
   --disable-ipv6 --allow-file-access-from-files --disable-java --disable-restore-session-state \
   --disable-sync --disable-translate --disk-cache-size=1 --overscroll-history-navigation=0 \
   --media-cache-size=1 \
   http://localhost:8080/stargazer-web
   ```

## Enable persistent logging

1. Execute these commands:

   ```bash
   mkdir -p /var/log/journal
   systemd-tmpfiles --create --prefix /var/log/journal
   systemctl restart systemd-journald.service
   ```

## BIOS settings

1. Power on the panel PC
1. Type ESC on boot to enter BIOS setup
1. Go to 'Chipset > South Bridge'
1. Change 'Restore AC Power Loss' to 'Power On'
1. Go to 'Save & Exit' to save the configuration

## Fix / add workaround c-state bug

1. Follow [these instructions](https://askubuntu.com/questions/803640/system-freezes-completely-with-intel-bay-trail/803649#803649)
