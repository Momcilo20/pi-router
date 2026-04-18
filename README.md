# pi-router
A Raspberry Pi router is a custom-built network device that replaces the cheap router your ISP hands you. Instead of relying on locked-down firmware with limited features, you gain complete control over routing, DNS, firewall rules, traffic shaping, and monitoring.


//
Step 1: Hardware You Need
The Raspberry Pi 4 Model B is the recommended choice. Its Gigabit Ethernet port and USB 3.0 bus make it capable of real-world routing speeds. The Pi 5 is also excellent but costs more for limited extra benefit in this use case.

Required Components
Raspberry Pi 4 (2 GB+) – Main compute board. 2 GB RAM is sufficient.
MicroSD Card (16 GB+) – Class 10 / A1 or faster. Samsung Endurance Pro recommended.
USB to Gigabit Ethernet Adapter – Second NIC for LAN. ASIX AX88179 chipset recommended.
USB-C Power Supply (3 A) – Official Raspberry Pi power supply preferred.
Two Ethernet Cables (Cat 5e+) – One for WAN, one for LAN / switch.
Case with Cooling – Aluminum passive or active fan case for 24/7 operation.
Important: The Pi 4 has one built-in Gigabit Ethernet port. You need a second NIC for the LAN side. The cleanest solution is a USB 3.0 to Gigabit Ethernet adapter. Avoid cheap RTL8152-based adapters since they cap out at USB 2.0 speeds (~300 Mbps).
//
Step 2: Installing the Operating System
Use Raspberry Pi OS Lite (64-bit). The Lite image has no desktop environment, which means less RAM usage and a smaller attack surface. Download the Raspberry Pi Imager from raspberrypi.com/software and flash the image to your MicroSD card.

Flashing with Raspberry Pi Imager
Open Raspberry Pi Imager and click Choose OS. Select “Raspberry Pi OS (other)” then “Raspberry Pi OS Lite (64-bit)”.
Click the gear icon to open advanced settings. Set a hostname (e.g. pirouter), enable SSH, and set your username and password.
Select your MicroSD card under Choose Storage, then click Write.
Insert the card into your Pi, connect a monitor and keyboard for the initial setup, and power it on.
Log in with the credentials you set in the imager and run a full update before continuing.

--
# Update all packages on fresh install
sudo apt update && sudo apt full-upgrade -y
sudo reboot
--

Security first: Change the default password immediately if you did not set one in the imager. Run passwd after first login. You should also disable password authentication for SSH and use key-based auth before putting this device on a live network.

//
Step 3: Configuring Network Interfaces
Before configuring anything, identify your network interfaces. Plug in your USB Ethernet adapter and run the following command:

--
ip link show
--
You will see output listing your interfaces. The built-in Ethernet is typically --eth0--. Your USB adapter will appear as something like eth1 or enx001122334455. Make note of both names. In this guide:

eth0 is the WAN interface (connects to your modem or ISP)
eth1 is the LAN interface (connects to your switch or directly to devices)
Assigning a Static IP to the LAN Interface
The LAN interface needs a static IP address because it will act as the default gateway for all your devices. Edit the DHCP client configuration file:

--
sudo nano /etc/dhcpcd.conf
--
Append the following lines at the bottom of the file to assign a static IP to eth1:

--
# Static IP for LAN interface
interface eth1
static ip_address=192.168.1.1/24
static domain_name_servers=1.1.1.1 8.8.8.8
--
Save the file with Ctrl+O, exit with Ctrl+X, and restart the DHCP client service:

--
sudo systemctl restart dhcpcd
--
//
Step 4: Setting Up DHCP and DNS with dnsmasq
dnsmasq is a lightweight DNS forwarder and DHCP server perfectly suited for a small network router. It will hand out IP addresses to your devices and cache DNS queries for faster browsing.


Installing dnsmasq

--
sudo apt install dnsmasq -y
--
Before editing the configuration, back up the original file:

--
sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.bak
--
Now create a new clean configuration file:

--
sudo nano /etc/dnsmasq.conf
--
Paste the following configuration:

--
# Interface to listen on (LAN side only)
interface=eth1
bind-interfaces
--
# DHCP range: assign IPs from .100 to .200
# Lease time: 24 hours
dhcp-range=192.168.1.100,192.168.1.200,24h
--
# Set the default gateway and DNS for clients
dhcp-option=3,192.168.1.1
dhcp-option=6,192.168.1.1
--
# Upstream DNS servers (Cloudflare + Google)
server=1.1.1.1
server=8.8.8.8
--
# Cache 1000 DNS entries
cache-size=1000
--
# Block DNS rebinding attacks
stop-dns-rebind
rebind-localhost-ok
--
# Log queries for debugging (optional)
# log-queries
--
Enable and start the dnsmasq service:

sudo systemctl enable dnsmasq
--
sudo systemctl start dnsmasq
--
sudo systemctl status dnsmasq
--
Tip: To assign a permanent reserved IP address to a specific device, add a line like dhcp-host=aa:bb:cc:dd:ee:ff,192.168.1.50 using the device’s MAC address. This is useful for servers, printers, and smart home hubs.
//

Step 5: Enabling NAT and IP Forwarding
IP forwarding allows the kernel to forward packets between interfaces. NAT (specifically masquerade) rewrites the source IP of outbound packets so they appear to come from the Pi rather than from individual LAN devices. This is how your modem/ISP sees a single IP while dozens of devices share the connection.

Enable IP Forwarding
Edit the kernel parameters file to enable forwarding at boot:

sudo nano /etc/sysctl.conf
-
Find and uncomment (or add) this line:

net.ipv4.ip_forward=1
-
Apply the change immediately without rebooting:

sudo sysctl -w net.ipv4.ip_forward=1
-
Configure NAT with iptables
Add the NAT masquerade rule so packets leaving through eth0 (WAN) have their source IP rewritten:

# Add NAT masquerade rule for WAN interface
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
-
# Allow established and related connections back in
sudo iptables -A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT
-
# Allow all forwarding from LAN to WAN
sudo iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT
-

Make iptables Rules Persistent
By default, iptables rules are lost on reboot. Install iptables-persistent to save them:

sudo apt install iptables-persistent -y
=
# When prompted, say YES to save current IPv4 rules
sudo netfilter-persistent save
-

//
Step 6: Configuring the Firewall
A proper firewall policy drops all unexpected inbound traffic on the WAN interface while allowing traffic you initiate from inside the network. This is a critical step before connecting your Pi router to the internet.

# Set default policies
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT ACCEPT

# Allow loopback interface
sudo iptables -A INPUT -i lo -j ACCEPT

# Allow established/related connections
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow SSH from LAN (do this BEFORE locking down)
sudo iptables -A INPUT -i eth1 -p tcp --dport 22 -j ACCEPT

# Allow DHCP from LAN clients
sudo iptables -A INPUT -i eth1 -p udp --dport 67:68 -j ACCEPT

# Allow DNS from LAN clients
sudo iptables -A INPUT -i eth1 -p udp --dport 53 -j ACCEPT
sudo iptables -A INPUT -i eth1 -p tcp --dport 53 -j ACCEPT

# Allow ICMP (ping) from LAN
sudo iptables -A INPUT -i eth1 -p icmp -j ACCEPT

# Allow forwarding: LAN to WAN
sudo iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT
sudo iptables -A FORWARD -i eth0 -o eth1 -m state --state ESTABLISHED,RELATED -j ACCEPT

# Save rules
sudo netfilter-persistent save
Test before rebooting! From a device connected to the LAN side, confirm you can SSH into the Pi at 192.168.1.1 and that you can reach the internet before rebooting. If something goes wrong, you can still access the Pi via a directly connected keyboard and monitor.

Verifying Your Rules
# List all rules with line numbers
sudo iptables -L -v --line-numbers

# Check NAT table
sudo iptables -t nat -L -v
Step 7 (Optional): Setting Up a Wi-Fi Access Point
The Raspberry Pi 4 has a built-in Wi-Fi chip (802.11ac). You can use it as a wireless access point, meaning devices can connect to your Pi router over Wi-Fi rather than Ethernet. We will use hostapd for this.

Install hostapd
sudo apt install hostapd -y
sudo systemctl unmask hostapd
sudo systemctl enable hostapd
Configure hostapd
sudo nano /etc/hostapd/hostapd.conf
# Use the on-board Wi-Fi interface
interface=wlan0
driver=nl80211

# Network name (SSID)
ssid=PiRouter_Network

# 5 GHz band (use hw_mode=g for 2.4 GHz)
hw_mode=a
channel=36
ieee80211n=1
ieee80211ac=1

# WPA2 security
wpa=2
wpa_passphrase=YourSecurePassword123
wpa_key_mgmt=WPA-PSK
rsn_pairwise=CCMP
Tell hostapd where to find its config file:

sudo nano /etc/default/hostapd
# Find and set:
DAEMON_CONF="/etc/hostapd/hostapd.conf"
Add wlan0 to dnsmasq
Add the Wi-Fi interface to your dnsmasq configuration so wireless clients also get DHCP leases. Open /etc/dnsmasq.conf and add:

interface=wlan0
dhcp-range=192.168.2.100,192.168.2.200,24h
dhcp-option=3,192.168.2.1
dhcp-option=6,192.168.1.1
Assign a static IP to wlan0 in /etc/dhcpcd.conf:

interface wlan0
static ip_address=192.168.2.1/24
nohook wpa_supplicant
Also add iptables rules to forward Wi-Fi traffic through the WAN interface:

sudo iptables -A FORWARD -i wlan0 -o eth0 -j ACCEPT
sudo iptables -A FORWARD -i eth0 -o wlan0 -m state --state ESTABLISHED,RELATED -j ACCEPT
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
sudo netfilter-persistent save
Start all services:

sudo systemctl start hostapd
sudo systemctl restart dnsmasq
sudo systemctl restart dhcpcd
