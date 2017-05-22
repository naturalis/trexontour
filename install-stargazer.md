# Installation procedure for Stargazer

## Install Ubuntu
1. Download Ubuntu server
2. Make a startup disk:
   `sudo if=ubuntu-16.04.2-server-amd64.iso of=/dev/sdX bs=1M`
3. Boot USB disk
4. Choose primary network interface: enp1s0
5. Hostname: stargazer-001.ontour.naturalis.nl
6. Full name for the new user: Ubuntu
7. Username for your account: ubuntu
8. Password for the new user: zie standaard wachtwoord Keepass
9. Encryption: no
10. Time zone: Europe/Amsterdam
11. Partitioning method: Guided - use entire disk and set up LVM
12. Select disk to partition: sda
13. Amount of volume group to use for guided partitioning: 100%
14. HTTP proxy information: <blank>
15. How do you want to manage upgrades on this system: Install security updates automatically
16. Choose software to install:
    * standard system utilities
    * OpenSSH server
17. Install the GRUB boot loader to the master boot record: Yes
18. Device for boot loader installation: /dev/sda

## Configure network

Based on this HOWTO configure the network settings:
1. Install firewalld and dnsmasq:
  `sudo apt install firewalld dnsmasq`
2. Create the network config directory for systemd-networkd:
   `mkdir /etc/systemd/network`
3. Add the configuration for the primary interface `enp1s0.network`:
   ```
   [Match]
   Name=enp1s0

   [Network]
   Description=Public interface
   DHCP=yes
   IPForward=yes
   ```
4. Add the configuration for the LAN interface `enp2s0.network`:
   ```
   [Match]
   Name=enp2s0

   [Network]
   Description=LAN interface
   Address=172.16.61.1/24
   IPForward=yes
   ```
5. Disable other networking services and enable systemd-networkd:
   ```
   systemctl disable network
   systemctl disable NetworkManager
   systemctl enable systemd-networkd
   ```
6. Also, let’s be sure to use systemd-resolved to handle our /etc/resolv.conf:
   ```
   systemctl enable systemd-resolved
   systemctl start systemd-resolved
   rm -f /etc/resolv.conf
   ln -s /run/systemd/resolve/resolv.conf /etc/resolv.conf
   ```
7. Enable and start dnsmasq:
   ```
   systemctl enable dnsmasq
   systemctl start dnsmasq
   ```

8. Enable masquerading on both interfaces, trust all internal traffic and allow access to web interface:
   ```
   firewall-cmd --zone=public --add-interface=enp1s0
   firewall-cmd --zone=public --add-masquerade
   firewall-cmd --zone=trusted --add-interface=enp2s0
   firewall-cmd --zone=trusted --add-masquerade
   firewall-cmd --zone=public --remove-service=dns
   firewall-cmd --zone=public --add-port=8080/tcp
   firewall-cmd --zone=public --add-port=5000/tcp
   firewall-cmd --runtime-to-permanent
   ```

9. Enable firewalld:
   `systemctl enable firewalld`

## Install docker
1. Follow the [instructions for installation of Docker CE on Ubunu](https://store.docker.com/editions/community/docker-ce-server-ubuntu)
2. Configure Docker to start on boot:
   `sudo systemctl enable docker`

## Install docker containers
1. Clone the stargazer repo:
   ```
   git clone https://github.com/MakeExpose/stargazer.git
   cd stargazer
   ```
2. Copy and edit the environment file:
   ```
   cp .env.example .env
   vim .env
   ```
3. Move the VPN configuration, key and secret to the configured directory.

4. Copy the systemd unit files to the right location:
   ```
   sudo cp *.(timer|service) /etc/systemd/system/
   systemctl enable stargazer-api.service stargazer-web.service site-to-site-vpn.service site-to-site-vpn.timer
   sudo systemctl daemon-reload
   ```

4. Start the docker containers:
   `systemctl start stargazer-api.service stargazer-web.service site-to-site-vpn.service site-to-site-vpn.timer`

## Install kiosk browser
1. Install packages:
   `sudo apt -y install xorg openbox build-essential`
2. Add kiosk user:
   `adduser kiosk`
3. Add this content to `/home/kiosk/.profile` to start X:
   ```
   # Startx Automatically
   . startx -- -nocursor
   logout
   ```
4. Autologin kiosk user:
   `mkdir /etc/systemd/system/getty@tty1.service.d`
   And add this to `/etc/systemd/system/getty@tty1.service.d/override.conf`:
   ```
   [Service]
   ExecStart=
   ExecStart=-/sbin/agetty -a kiosk --noclear %I $TERM
   ```
5. Install Chromium
   `sudo apt -y install chromium-browser`

6. Add autostart config file:
   ```
   sudo su kiosk
   mkdir /home/kiosk/.config/openbox
   vim /home/kiosk/.config/openbox/autostart.sh
   chmod 644 /home/kiosk/.config/openbox/autostart.sh
   ```
   And add these lines:
   ```
   sed -i 's/"exited_cleanly":false/"exited_cleanly":true/' ~/.config/chromium/'Local State'
   sed -i 's/"exited_cleanly":false/"exited_cleanly":true/; s/"exit_type":"[^"]\+"/"exit_type":"Normal"/' ~/.config/chromium/Default/Preferences
   /bin/sleep 25
   chromium-browser --disable-infobars --disable-session-crashed-bubble \
   --kiosk --enable-kiosk-mode --enabled --touch-events --touch-events-ui \
   --disable-ipv6 --allow-file-access-from-files --disable-java --disable-restore-session-state \
   --disable-sync --disable-translate --disk-cache-size=1 --overscroll-history-navigation=0 \
   --media-cache-size=1 \
   http://localhost:8080/stargazer-web
   ```

## Enable persistent logging

1. Execute these commands:
   ```
   mkdir -p /var/log/journal
   systemd-tmpfiles --create --prefix /var/log/journal
   systemctl restart systemd-journald.service
   ```

## BIOS settings

1. Power on the panel PC
2. Type ESC on boot to enter BIOS setup
3. Go to 'Chipset > South Bridge'
4. Change 'Restore AC Power Loss' to 'Power On'
5. Go to 'Save & Exit' to save the configuration

## Fix / add workaround c-state bug

1. Follow [these instructions](https://askubuntu.com/questions/803640/system-freezes-completely-with-intel-bay-trail/803649#803649)
