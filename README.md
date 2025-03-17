# Setting Up a Wi-Fi Hotspot with Captive Portal on Raspberry Pi

This guide will walk you through setting up a Wi-Fi hotspot with a captive portal on a Raspberry Pi. The captive portal will redirect all HTTP traffic to an informational page (index.html) regardless of internet availability. We will use hostapd for the access point, dnsmasq for DHCP/DNS, nginx for the web server, and iptables for traffic management.

## Prerequisites
- Raspberry Pi with Raspberry Pi OS installed.
- A Wi-Fi adapter that supports Access Point (AP) mode.
- Terminal access (via SSH or locally).
- Optional: Ethernet connection for internet access.

## Step 1: Update the System
Ensure all packages are up-to-date.

```bash
sudo apt update && sudo apt upgrade -y
```

## Step 2: Install Required Packages
Install packages necessary for the access point, DNS/DHCP, web server, and iptables rules persistence.

```bash
sudo apt install -y hostapd dnsmasq nginx iptables-persistent
```
During installation, select "Yes" when prompted to save IPv4/IPv6 rules.

## Step 3: Configure Wi-Fi Access Point with hostapd

Create the hostapd configuration file:

```bash
sudo nano /etc/hostapd/hostapd.conf
```
Add the following content:

```text
interface=wlan0
driver=nl80211
ssid=MyPiAP
hw_mode=g
channel=7
wmm_enabled=0
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_passphrase=YourPassword123
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
```

Specify the configuration file path:

```bash
sudo nano /etc/default/hostapd
```
Modify or add the following line:

```text
DAEMON_CONF="/etc/hostapd/hostapd.conf"
```

Enable and start hostapd:

```bash
sudo systemctl unmask hostapd
sudo systemctl enable hostapd
sudo systemctl start hostapd
```

## Step 4: Configure DHCP and DNS with dnsmasq

Create the dnsmasq configuration file:

```bash
sudo nano /etc/dnsmasq.conf
```

Add or modify the following lines:

```text
interface=wlan0
listen-address=192.168.4.1
bind-interfaces
domain-needed
bogus-priv
dhcp-range=192.168.4.100,192.168.4.200,24h
```

Enable and start dnsmasq:

```bash
sudo systemctl enable dnsmasq
sudo systemctl start dnsmasq
```

## Step 5: Assign Static IP to wlan0

Edit the dhcpcd configuration:

```bash
sudo nano /etc/dhcpcd.conf
```

Append the following lines:

```text
interface wlan0
static ip_address=192.168.4.1/24
nohook wpa_supplicant
nohook resolv.conf
```

Restart dhcpcd:

```bash
sudo systemctl restart dhcpcd
```

## Step 6: Prevent /etc/resolv.conf from Being Overwritten

Create a custom resolv.conf file:

```bash
sudo nano /etc/resolv.conf
```

Add the following:

```text
nameserver 8.8.8.8
nameserver 8.8.4.4
```

Make the file immutable:

```bash
sudo chattr +i /etc/resolv.conf
```

To edit later:

```bash
sudo chattr -i /etc/resolv.conf
```

## Step 7: Configure nginx for Captive Portal

Create the page:

```bash
sudo mkdir -p /var/www/html
sudo nano /var/www/html/index.html
```

Add the following content:

```html
<!DOCTYPE html>
<html>
<head><title>Welcome to My Hotspot</title></head>
<body>
    <h1>Welcome to My Hotspot</h1>
    <p>You are connected to the MyPiAP network. Click "Authorize" to proceed.</p>
    <form action="/accept" method="POST">
        <button type="submit">Authorize</button>
    </form>
</body>
</html>
```

Modify nginx configuration:

```bash
sudo nano /etc/nginx/sites-available/default
```

Replace the content with:

```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    root /var/www/html;
    index index.html;
    server_name _;
    
    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

Restart nginx:

```bash
sudo nginx -t
sudo systemctl restart nginx
```

## Step 8: Configure iptables for Traffic Management

Flush existing rules:

```bash
sudo iptables -F
sudo iptables -t nat -F
```

Add new rules:

```bash
sudo iptables -A FORWARD -i wlan0 -p udp --dport 53 -j ACCEPT
sudo iptables -A FORWARD -i wlan0 -p tcp --dport 53 -j ACCEPT
sudo iptables -A FORWARD -i wlan0 -p tcp --dport 80 -j ACCEPT
sudo iptables -A FORWARD -i wlan0 -p tcp --dport 443 -j ACCEPT

sudo iptables -t nat -A PREROUTING -i wlan0 -p tcp --dport 80 -j DNAT --to-destination 192.168.4.1:80
sudo iptables -A INPUT -i wlan0 -p tcp --dport 80 -j ACCEPT

sudo iptables -t nat -A PREROUTING -i wlan0 -p udp --dport 53 -j DNAT --to-destination 192.168.4.1

sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

Enable IP forwarding:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

Make it persistent:

```bash
sudo nano /etc/sysctl.conf
```

Ensure the following line is present:

```text
net.ipv4.ip_forward=1
```

Apply the changes:

```bash
sudo sysctl -p
```

Save iptables rules:

```bash
sudo netfilter-persistent save
sudo netfilter-persistent reload
```

