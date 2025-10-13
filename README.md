# Raspberry_Pi_Wi-Fi_Hotspot_with_VPN
Raspberry Pi 4 Wi-Fi Hotspot with VPN
Since Raspberry Pi OS Bookworm, the network setting is managed by NetworkManager (instead of dhcpcd). It will take control for what was done by hostapd and dnsmasq as well.

1. Prerequisites:

  - Raspberry Pi 4B 4GB
  - Private OpenVPN file (e.g., client.ovpn)
  - Another computer
  - SD Card and adaptor
  - LAN between Router and RPi

2. Raspberry Pi Image Setting before flash:
    - Raspberry Pi OS (64Bit) Trixie without desktop
    - Advanced options:
      - Set hostname
      - Enable SSH
      - Username and Password set

3. Unblock the Wi-Fi (if message shows)
   
   Wi-FI of RPi might be blocked without Country setting
   Upon Login, the prompt might shows this message:
   ```
   Wi-Fi is currently blocked by rfkill.
   Use raspi-config to set the country before use.
   ```
   - Run `sudo raspi-config`
   - Navigate to Localisation Options > L4 WLAN Country
   - Select your country, e.g., GB Britain (UK)

4. Install Required Packages   
  ```
  sudo apt update
  sudo apt upgrade -y
  sudo apt-get install vim openvpn iptables -y
  ```

5. Configure OpenVPN (for Private VPN)

  - Transfer OpenVPN file to the RPi:
      ```
      scp client.ovpn username@192.168.XXX.XXX:~/client.conf
      ```
  - Connect to the RPi via SSH
  - Move OpenVPN file
      ```
      sudo mv client.conf /etc/openvpn/client.conf
      ```
  - Create an auth file by

      `sudo vim /etc/openvpn/client.auth`

    with these 2 lines
    
      ```
      username
      password
      ```
  - add permission setting for a bit of security
      ```
      sudo chmod 600 /etc/openvpn/client.auth
      ```
  - Update client.conf by
    
      `sudo vim /etc/openvpn/client.conf`
    
    with
    
      ```
      ...
      auth-user-pass /etc/openvpn/client.auth
      ...
      tls-cipher "DEFAULT:@SECLEVEL=0" <-- for older version
      ...
      ```
    
  - Autostart the client at boot by
    
       `sudo vim /etc/default/openvpn`
    
    with

      ```
      ...
      autostart="client" 
      ...
      ```

  - Reboot and verify to see if tun0 is active
    ```
    ip addr show
    ```

6. Configure Wi-Fi Hotspot
   - Create the hotspot - NEED TO MODIFY

     ```
     sudo nmcli con add type wifi ifname wlan0 con-name hotspot autoconnect yes ssid <<<SSID>>>
     ```
     
   * No quote required for SSID
   - Configure hotspot - NEED TO MODIFY

     ```
     sudo nmcli con modify hotspot 802-11-wireless.mode ap 802-11-wireless.band bg ipv4.method shared wifi-sec.key-mgmt wpa-psk wifi-sec.psk "<<<PASSWORD>>>"
     ```
     * quote REQUIRED for PASSWORD
     * “ipv4.method shared” tells NM to:   
     *   • assign 10.42.x.x to wlan0 automatically   
     *   • enable internal DHCP + NAT using its built-in dnsmasq
     
   - Start the hotspot
     
     `sudo nmcli con up hotspot`
  
   - VPN Routing
     
     ```
     sudo iptables -t nat -A POSTROUTING -o tun0 -j MASQUERADE
     sudo iptables -A FORWARD -i tun0 -o wlan0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
     sudo iptables -A FORWARD -i wlan0 -o tun0 -j ACCEPT
     ```
   - Double check
     
     ```
     sudo iptables -t nat -L -n -v
     ...
     Chain POSTROUTING (policy ACCEPT 0 packets, 0 bytes)
     pkts bytes target     prot opt in  out   source      destination
        0     0 MASQUERADE  all  --  *   tun0  0.0.0.0/0  0.0.0.0/0

     sudo iptables -L FORWARD -n -v
     Chain FORWARD (policy DROP)
     pkts bytes target     prot opt in   out   source      destination
        0     0 ACCEPT     all  --  tun0 wlan0  0.0.0.0/0  0.0.0.0/0  ctstate RELATED,ESTABLISHED
        0     0 ACCEPT     all  --  wlan0 tun0  0.0.0.0/0  0.0.0.0/0

     sudo iptables -t nat -L -n -v
     sudo iptables -L FORWARD -n -v
     pkts bytes target     prot opt in   out   source      destination
        123  9087 MASQUERADE  all  --  *  tun0  0.0.0.0/0  0.0.0.0/0

     ip route show
     10.8.0.0/24 dev wlan0 proto kernel scope link src 10.8.0.1
     ```

7. Verificaton & Finish Up
   - Connect a device and check IP
   - Save the rules for persistsence if working
    ```
    sudo apt install iptables-persistent -y
    sudo netfilter-persistent save
    ```
   - Reboot and try again

- Something went wrong?
  - Cleanup / Replace Existing iptables Reules:
    ```
    sudo iptables -F
    sudo iptables -t nat -F
    sudo iptables -X
    ```
  

