

# VPN Setup Starting From Base Raspbian Images

## 1. Install Raspbian

Install Raspbian Stretch on the Pi's in the usual way. Eventually the client needs to be configured 
to boot headless with VNC desktop and it's probably good to set up both Pi's that way from the 
beginning. Use a wired ethernet connection to the server Pi so that its eth0 interface has a known 
address on the local network. The initial network connection to the client Pi can be ethernet or 
the on-board wifi - later both of those will be reconfigured as separate private networks that 
portable devices can connect to. The final network setup looks like:


* 10.200.200.0/24 subnet for the tunnel network between the client and server.
* 10.200.200.1 for the server tunnel interface (wg0)
* 10.200.200.2 for the client tunnel interface (wg0)
* 10.100.100.0/24 subnet for the wireless network client hosts on wlan0 (on-board wifi).
* 10.100.100.1 for the client wireless network interface (wlan0)
* 10.150.150.0/24 subnet for the wired network client hosts on eth0.
* 10.150.150.1 for the client wired network interface (eth0)
* Public wifi DHCP assigns IP for Client Pi dongle interface (eth1)

Use terminals on the VNC desktops for the following steps.



## 2. Install WireGuard on both Pi's

This is adapted from wireguard.com/install, "Option B) Compiling from Source", for Debian

Install linux header files for Raspbian:

```
sudo apt-get update && sudo apt-get upgrade
sudo apt-get install raspberrypi-linux-headers
sudo reboot
```

Check that kernel version from "uname -r" matches directory in /lib/modules. Otherwise the build is 
likely to fail. 

Install some additional libraries:

```
sudo apt-get update
sudo apt-get install libmnl-dev libelf-dev build-essential pkg-config
```

Get the current WireGuard snapshot and build it:

```
wget https://git.zx2c4.com/WireGuard/snapshot/WireGuard-x.x.x.tar.xz
tar xvf /WireGuard-x.x.x.tar.xz

cd WireGuard/src
make
sudo make install
```

Check that the wg and wg-quick commands are installed



## 3. Generate WireGuard encryption keys for both Pi's

On the server run:

```
wg genkey | tee server_private_key | wg pubkey > server_public_key
```

The server public key value will need to be transferred to the client.

On the client run:

```
wg genkey | tee client_private_key | wg pubkey > client_public_key
```

The client public key value will be need to be transferred to the server.



## 4. Server-specific configuration

### 4.1 Install utility packages

```
sudo apt update && sudo apt-get upgrade
sudo apt-get install dnsutils iptables-persistent
```


### 4.2 WireGuard config file

Create file /etc/wireguard/wg0.conf containing:

```
[Interface]
Address = 10.200.200.1/32
DNS = 10.200.200.1
ListenPort = 51820
PrivateKey = <server-private-key>
MTU = 1500
SaveConfig = true

[Peer]
PublicKey = <client-public-key>
AllowedIPs = 10.200.200.2/32
```

replacing server-private-key and client-public-key with the values generated previously.

See the wg-quick and wg man pages for info on the config file entries. The optional MTU parameter 
is included to try to avoid potential fragmentation errors during packet transmission.

Ensure that only root can access the file:

```
sudo chown -v root:root /etc/wireguard/wg0.conf
sudo chmod -v 600 /etc/wireguard/wg0.conf
```


### 4.3 Bring up WireGuard

```
sudo wg-quick up wg0
sudo systemctl enable wg-quick@wg0.service
```

Check that ifconfig shows a new interface named wg0. WireGuard will start automatically on boot. 
It's not normally necessary to shut it down but the command to do so is:

```
sudo wg-quick down wg0
```


### 4.4 Enable IP forwarding

Edit file /etc/sysctl.conf and uncomment or add the line:

```
net.ipv4.ip_forward=1
```

Enable the change:

```
sudo -s
sysctl -p
echo 1 > /proc/sys/net/ipv4/ip_forward
ctrl-d
```


### 4.5 Configure firewall rules

Allow loopback connections:

```
sudo iptables -A INPUT -i lo -j ACCEPT
```

Allow established connections:

```
iptables -A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
```

Allow ssh and VNC connections:

```
sudo iptables -A INPUT -p tcp -m tcp --dport 22 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
sudo iptables -A INPUT -p tcp -m tcp --dport 5900 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
```

Allow VPN connections on WireGuard port:

```
sudo iptables -A INPUT -p udp -m udp --dport 51820 -m conntrack --ctstate NEW -j ACCEPT
```

Allow DNS connections from clients through the VPN tunnel:

```
sudo iptables -A INPUT -s 10.200.200.0/24 -p tcp -m tcp --dport 53 -m conntrack --ctstate NEW -j ACCEPT
sudo iptables -A INPUT -s 10.200.200.0/24 -p udp -m udp --dport 53 -m conntrack --ctstate NEW -j ACCEPT
```

Allow forwarding from the tunnel:

```
sudo iptables -A FORWARD -i wg0 -m conntrack --ctstate NEW -j ACCEPT
```

Enable NAT for packets leaving the server from the tunnel:

```
sudo iptables -t nat -A POSTROUTING -s 10.200.200.0/24 -o eth0 -j MASQUERADE
```

Set default policies:

```
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT ACCEPT
```

Make the rules persistent:

```
systemctl enable netfilter-persistent
netfilter-persistent save
```


### 4.6 Install the DNS server

This setup uses the unbound DNS resolver, as recommended in www.ckn.io, 
"WireGuard VPN: Typical Setup". DNS requests by devices connected to the client 
Pi will be forwarded to this resolver through the tunnel, preventing the leakage 
of information from the devices.

Begin by installing the packages:

```
sudo apt-get install unbound unbound-host
```

Fetch the list of root servers:

```
sudo curl -o /var/lib/unbound/root.hints https://www.internic.net/domain/named.cache
```

Replace the contents of the configuration file /etc/unbound/unbound.conf with:

```
server:

  num-threads: 4

  #Enable logs
  verbosity: 1

  #list of Root DNS Server
  root-hints: "/var/lib/unbound/root.hints"

  #Use the root servers key for DNSSEC
  auto-trust-anchor-file: "/var/lib/unbound/root.key"

  #Respond to DNS requests on all interfaces
  interface: 0.0.0.0
  max-udp-size: 3072

  #Authorized IPs to access the DNS Server
  access-control: 0.0.0.0/0 refuse

  access-control: 127.0.0.1                 allow
  access-control: 10.200.200.0/24         allow

  #not allowed to be returned for public internet  names
  private-address: 10.200.200.0/24

  # Hide DNS Server info
  hide-identity: yes
  hide-version: yes

  #Limit DNS Fraud and use DNSSEC
  harden-glue: yes
  harden-dnssec-stripped: yes
  harden-referral-path: yes

  #Add an unwanted reply threshold to clean the cache and avoid when possible a DNS Poisoning
  unwanted-reply-threshold: 10000000


  #Have the validator print validation failures to the log.
  val-log-level: 1

  #Minimum lifetime of cache entries in seconds
  cache-min-ttl: 1800 

  #Maximum lifetime of cached entries
  cache-max-ttl: 14400
  prefetch: yes
  prefetch-key: yes
```

Optionally append local hostname/address data to /etc/unbound/unbound.conf. It could be as simple as:

```
local-data: "myserver A 10.0.0.5"
```

Enable the service:

```
chown -R unbound:unbound /var/lib/unbound
systemctl enable unbound
```

Test the service:

```
sudo unbound-host -v  -C /etc/unbound/unbound.conf cloudflare.com
```

unbound supports DNSSEC.



# 5. Client-specific configuration

## 5.1 Install utility packages

```
sudo apt update && sudo apt-get upgrade
sudo apt-get install hostapd dnsmasq dnsutils
```


## 5.2 Configure wifi dongle

Insert the dongle into a USB slot.  The dongle will provide the connection to public wifi. 
Raspbian Stretch includes dhcpcd, a DHCP utility that is used to acquire an IP address for 
the dongle's wlan1 interface.

Append the following lines to /etc/dhcpcd.conf:

```
denyinterfaces wlan0
denyinterfaces eth0
interface wlan1
```

Check if /etc/wpa_supplicant/wpa_supplicant.conf contains an entry for the local wifi network. If not, edit the file to append an entry specifiying the network ssid and password:

```
network={
	ssid="network name"
	psk="network password"
	key_mgmt=WPA-PSK
}
```

Restart dhcpcd with the following command. The wlan0 and eth0 interfaces will be down so 
reconnect to VNC using the wlan1 interface. The IP address can be determined from the 
local router's UI or from apps like Fing.

```
sudo systemctl restart dhcpcd
```


## 5.3 Configure wlan0 and eth0

wlan0 and eth0 are the interfaces used by portable devices to connect to the Pi. The wlan0 wifi is 
managed by the hostapd utility. DHCP services for both networks are provided by the dnsmasq utility. 

The Stretch version of Raspbian introduced networking changes that cause problems for configuring 
these additional interfaces. Consequently the setup uses workarounds that may need to be revisited 
with future Raspbian updates.

Append interface definitions to /etc/network/interfaces:

```
auto wlan0
iface wlan0 inet static
    address 10.100.100.1
    netmask 24

allow-hotplug eth0
iface eth0 inet static
    address 10.150.150.1
    netmask 24
```

Replace the contents of /etc/hostapd/hostapd.conf with the following lines, substituting values 
for ssid and wpa_passphrase:

```
interface=wlan0
hw_mode=g
channel=11
ieee80211d=1
driver=nl80211
country_code=BZ
ieee80211n=1
wmm_enabled=1

ssid=<some network name>
auth_algs=1
wpa=2
wpa_key_mgmt=WPA-PSK
rsn_pairwise=CCMP
wpa_passphrase=<some password>
```

Change the wifi channel number if desired.  Note that this setup only supports a single channel.

Replace the contents of /etc/dnsmasq.conf with the lines:

```
dhcp-authoritative
read-ethers
bogus-priv
domain-needed

interface=wlan0
  listen-address=10.100.100.1
  dhcp-range=wlan0,10.100.100.50,10.100.100.100,12h

interface=eth0
  listen-address=10.150.150.1
  dhcp-range=eth0,10.150.150.50,10.150.150.100,12h

dhcp-option=option:dns-server,10.200.200.1
```

Insert the following line in /etc/rc.local before the "exit 0" line.  This is a workaround to 
bring up dhcpcd on boot since systemd will fail to start it when there are interface 
definitions present in /etc/network/interfaces.

```
dhcpcd
```


Start the wlan0 and eth0 networks:

```
sudo ifup wlan0
sudo service dnsmasq start
sudo service hostapd start
sudo update-rc.d hostapd enable
```

Check the setup by connecting to the VNC desktop over each network. To test the wifi network 
connect using the ssid and password chosen above and the addresses 10.100.100.1. To test the 
wired ethernet connection use the addres 10.150.150.1.

## 5.4 Update hostapd if needed

Check the version of hostapd installed earlier from the Raspbian/Debian repositories:

```
hostapd -v
```

If prior to 2.6 then a newer version should be installed to avoid a possible wifi issue which can cause a disconnect and need a reboot to clear. Getting the new version requires building hostapd from source. The existing install can be retained to provide the boot init scripts - only the hostapd binary will be replaced.The following steps are based on [this reference](https://wireless.wiki.kernel.org/en/users/documentation/hostapd).

```
sudo apt-get install libnl-dev libssl-dev

wget http://w1.fi/releases/hostapd-x.y.z.tar.gz
tar xzvf hostapd-x.y.z.tar.gz
cd hostapd-x.y.z/hostapd
cp defconfig .config
```

Edit .config and check that following line is uncommented:

```
#CONFIG_DRIVER_NL80211=y
```

Compile the source:

```
make
```

Test the new binary:

```
sudo systemctl down hostapd.service
sudo ./hostapd /etc/hostapd/hostapd.conf
```

Check that the wifi service is up and working. If yes then move or rename the previously install hostapd binary and replace it with the new version. Restart the service:

```
sudo systemctl up hostapd.service
```

## 5.5 Create client config file for WireGuard

Create file /etc/wireguard/wg0.conf containing:

```
[Interface]
Address = 10.200.200.2/32
PrivateKey = <client-private-key>
MTU = 1500
SaveConfig = true

[Peer]
PublicKey = <server-public-key>
AllowedIPs = 0.0.0.0/0
Endpoint = <server-ip>:51820
PersistentKeepalive = 21
```

replacing client-private-key and server-public-key with the values generated previously. 
Replace server-ip with the IP address of the eth0 interface on the server. During setup 
and testing when both Pi's are connected to the same network this address can just be the 
private network address as shown by the ifconfig command. Later it will need to be changed 
to a public IP address or domain.

Ensure that only root can access the file:

```
sudo chown -v root:root /etc/wireguard/wg0.conf
sudo chmod -v 600 /etc/wireguard/wg0.conf
```

See the wg-quick and wg man pages for info on the config file entries. The optional MTU 
parameter is included to try to avoid potential fragmentation errors during packet transmission.


## 5.6 Edit the wg-quick script

A change in the wq-quick script is needed in order to allow access to the server's local network 
from devices attached to the client Pi. For example, it might be desired to access a file 
or media server.

Make a backup copy of /usr/bin/wg-quick and then make the folowing two changes in 
function add_default().

- Delete the following line:

```
cmd ip $proto rule add not fwmark $DEFAULT_TABLE table $DEFAULT_TABLE
```

- Append the following line just after the remaining "ip $proto rule" commands:

```
cmd ip $proto rule add fwmark 2 table $DEFAULT_TABLE
```

This ensures that packets from wlan0 or eth0 that are intended for an address on the server's 
local network will be routed into the tunnel even if the client Pi's public network has 
conflicting addresses.

Note that in this setup only packets from the devices connected to the Pi are routed into
the VPN tunnel. Any packets created locally on the Pi will be routed to the local network
and beyond. This is not a problem if the Pi desktop is only used to get connected to the
public wifi. An alternative setup is possible in which the tunnel is the defautl route for
all packets, however in that case WireGuard must be started manually each time after
connecting to public wifi.


## 5.7 Bring up WireGuard

```
sudo wg-quick up wg0
sudo systemctl enable wg-quick@wg0.service
```

Check that ifconfig shows a new interface named wg0. 


## 5.8 Enable IP forwarding

Edit file /etc/sysctl.conf and uncomment or add the line:

```
net.ipv4.ip_forward=1
```

Enable the change:

```
sudo -s
sysctl -p
echo 1 > /proc/sys/net/ipv4/ip_forward
ctrl-d
```


## 5.9 Configure firewall rules

Allow loopback connections:

```
sudo iptables -A INPUT -i lo -j ACCEPT
```

Allow established connections:

```
sudo iptables -A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
```

Allow connections on wlan0 or eth0:

```
sudo iptables -A INPUT -i wlan0 -m conntrack --ctstate NEW -j ACCEPT
sudo iptables -A INPUT -i eth0 -m conntrack --ctstate NEW -j ACCEPT
```

Allow forwarding to the tunnel:

```
sudo iptables -A FORWARD -i wlan0 -o wg0 -m conntrack --ctstate NEW -j ACCEPT
sudo iptables -A FORWARD -i eth0 -o wg0 -m conntrack --ctstate NEW -j ACCEPT
sudo iptables -t mangle -A PREROUTING -p all -i wlan0 -j MARK --set-mark 2
sudo iptables -t mangle -A PREROUTING -p all -i eth0 -j MARK --set-mark 2
```

Enable NAT for packets entering the tunnel:

```
sudo iptables -t nat -A POSTROUTING -o wg0 -j MASQUERADE
```

Set default policies:

```
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT ACCEPT
```

Make the rules persistent:

```
sudo sh -c "iptables-save > /etc/iptables.ipv4.nat"
```

and Insert the following line in /etc/rc.local before the "exit 0" line:

```
iptables-restore < /etc/iptables.ipv4.nat
```



# 6. Test VPN tunnel locally

Connect a device to one of the client private interfaces and start a browser. If web access is working then as a sanity check confirm that there is traffic through the server's wg0 interface. This can be done simply on the server desktop with the command "ifconfig wg0", which will show packet transmitted/received counts. Alternatively, monitor the traffic level using "vnstat -l -i wg0". Also check that DNS requests are not leaking outside the tunnel by using a website like dsnleak.com.



# 7. Check security

If not already done, change to non-default passwords for ssh and vnc, at least on the client Pi. Check that there are no open ports on the client Pi public wifi side, using a utility like nmap.



# 8. Test the VPN tunnel remotely

For remote testing the server IP setting in the clients WireGuard config file must point to a public address or host and the WireGuard port on the server must be accessible for udp. Since ISP-assigned addresses normally do not change very often a workable method is to just use the numeric IP and forward the port on the local router. Alternatively a DDNS service can be used to get a fixed host name instead.

Now it should be possible to connect to the server from any wifi location. A portable device with a VNC viewer app is required to start the connection. The device can connect using either the wifi or ethernet private network. The wifi widget on the desktop is used to select the ssid and sign in. Captive portals will require starting a browser on the desktop to get access. 

Use a site like whoer.net to check that the ip address is the same as the WireGuard server. If the wifi interface is not needed it can be disabled/enabled with the commands:

```
sudo ifdown wlan1
sudo ifup wlan1
```

To avoid sdcard damage use the VNC desktop to shut down the Pi at the end of the session. 
