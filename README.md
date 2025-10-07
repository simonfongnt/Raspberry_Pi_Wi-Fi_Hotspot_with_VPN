# Raspberry_Pi_Wi-Fi_Hotspot_with_VPN
Raspberry Pi 4 Wi-Fi Hotspot with VPN

1. Initial Setup

  - Flash Raspberry Pi OS on the Pi 4.
  - Enable SSH for headless access by placing an empty ssh file in /boot.
  - Connect to Pi via SSH.
  - Private OpenVPN file (.ovpn -> .conf)

2. Install Required Packages
```
sudo apt update
sudo apt upgrade
sudo apt-get install openvpn iptables hostapd dnsmasq -y
sudo systemctl stop hostapd dnsmasq
```
X. Configure OpenVPN (Optional)
- Move OpenVPN file to folder
```
mv xXxXx.ovpn /etc/openvpn/xXxXx.conf
```
- Create an auth file /etc/openvpn/client.auth
```
username
password
```
- add permission setting for a bit of security
```
sudo chmod 600 /etc/openvpn/client.auth
```
- Update xXxXx.conf with `sudo vum /etc/openvpn/xXxXx.conf`
```
...
auth-user-pass /etc/openvpn/client.auth
...
tls-cipher "DEFAULT:@SECLEVEL=0" <-- for older version
...
```
- Autostart the client at boot with `sudo vim /etc/default/openvpn`
```
...
autostart="xXxXx" 
...
```
3. Configure Hostapd (Wi-Fi Access Point)
Create/edit /etc/hostapd/hostapd.conf with content (no blanket needed for ssid & passphrase):
```
interface=wlan0
driver=nl80211
ssid=YourSSID
hw_mode=g
channel=7
wmm_enabled=0
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_passphrase=YourStrongPassword
wpa_key_mgmt=WPA-PSK
rsn_pairwise=CCMP
```
Edit /etc/default/hostapd to specify config file path:
```
...
DAEMON_CONF="/etc/hostapd/hostapd.conf"
...
```
4. Configure Static IP for wlan0
Edit /etc/dhcpcd.conf and add at the bottom:
```
interface wlan0
    static ip_address=192.168.4.1/24
    nohook wpa_supplicant
```
  - This sets the hotspot IP to 192.168.4.1 and disables wpa_supplicant on wlan0 to avoid conflict.
5. Configure DHCP and DNS Server (dnsmasq)
  - Backup default config and create a new dnsmasq config:
```
sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.orig
sudo nano /etc/dnsmasq.conf
```
Add the following content:
```
interface=wlan0      # Use interface wlan0
dhcp-range=192.168.4.2,192.168.4.20,255.255.255.0,24h
```
  - This configures DHCP to hand out IPs in the range 192.168.4.2 to 192.168.4.20.
6. Enable IP Forwarding and Configure NAT (for VPN or Internet Sharing)
Enable IPv4 forwarding:
Edit /etc/sysctl.conf and uncomment:
```
net.ipv4.ip_forward=1
```
Apply immediately:
```
sudo sysctl -w net.ipv4.ip_forward=1
```
Enable routing to allow traffic flow (Old, `sudo vim /etc/sysctl.d/routed-ap.conf`)
```
# Enable IPv4 routing
net.ipv4.ip_forward=1
```
Set up NAT with iptables (example):
```
sudo iptables -t nat -A POSTROUTING -o tun0 -j MASQUERADE
sudo iptables -A FORWARD -i tun0 -o wlan0 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i wlan0 -o tun0 -j ACCEPT

sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
sudo sh -c "iptables-save > /etc/iptables.ipv4.nat"
```
Make iptables rules persistent (optional):
Install iptables-persistent or add a script to load on boot.
Or, save it to be loaded on boot time by
```
sudo netfilter-persistent save
```
Ensure Wifi Radio is not blocked on Raspberry Pi
```
sudo rfkill unblock wlan
```
7. Enable and Start Services
```
sudo systemctl unmask hostapd
sudo systemctl enable hostapd
sudo systemctl enable dnsmasq
sudo systemctl start hostapd
sudo systemctl start dnsmasq
```
8. Verify Everything
- Check hostapd status:
```
sudo systemctl status hostapd
```
- Check dnsmasq status:
```
sudo systemctl status dnsmasq
```
- Check IP address on wlan0:
```
ip addr show wlan0
```
Should show 192.168.4.1.
- Connect a client device to the Wi-Fi SSID and check if it gets an IP in the DHCP range.



Possible Issues:
- Fix Wi-Fi Soft Block (rfkill)
  - Run:
```
sudo rfkill unblock wlan

```
  - To make this permanent, set the Wi-Fi country:
```
sudo raspi-config
```
  - Go to Localisation Options > Wi-Fi Country and select your country.
  - Reboot.
