# WireGuard Portable VPN Client and Server on Raspberry Pi 3's

![alt text](https://github.com/jimb245/2PiWireGuardVPN/blob/master/2PiVPN.jpg)

This post describes setting up a portable WireGuard VPN access point and server on Raspberry Pi 3's. The shortcut method is to start from the provided Raspbian-based image files. They contain all the necessary software so setup consists mainly of installing the images on the Pi's and editing some parameters like passwords and IP addresses. It's good to also update WireGuard to the current snapshot since it is still regarded as experimental by its authors. For reference the setup procedure starting from the base Rasbian images is [here](InstallFromRaspbianBase.md). For general information about WireGuard see References 1-2 below. References 3-5 are install guides for some WireGuard configurations on Ubuntu and Raspbian.

To use this 2-Pi VPN the server is connected with an ethernet cable to your home network, or potentially any other protected network with internet access. UDP packets from the internet must be able to reach the WireGuard port on the server so the port needs to be forwarded on the local router. If you happen to be running an openwrt router then another minimalist alternative would be to install WireGuard on the router itself. 

The portable access point is similar to a travel router, with both public and private interfaces. You connect the public interface to a public wifi and connect your portable devices to the private interface, using either wifi or an ethernet cable. In practice you first connect to the private interface with a portable device that has a VNC client. Using the VNC desktop you connect the Pi to public wifi and if necessary start a browser in order to handle captive portals. At the end of the session the desktop can be used to safely shutdown the Pi to avoid potentially corrupting the sdcard.

In this setup the VPN is the "routed" type, not "bridged". The portable devices can reach the internet and the server's local network, but not the reverse. Systems on the server's local network can be addressed by numeric address, or by hostname if name entries are added to the server's configuration.

The WireGuard developers are working on cross-platform apps so the usefulness of the access point approach versus apps depends on how many devices are connecting and what apps are available. The access point does provide an extra layer of isolation and updatable security.

Of course installing the server on a home network won't help if the goal is to prevent snooping by an ISP. For that you could consider using a cloud server. Reference 3. below explains how to set up a WireGuard server on a Ubuntu system, for example. The portable access point could be used with either server. On the other hand having a server with your home IP can help with some services that try to block VPN's or impose geographic restrictions.

Software versions on the images:
- Raspbian Stretch (March 2018)
- WireGuard 0.0.20180304

## What You Will Need

- Two Raspberry Pi 3's with cases
- Two 8GB microSD cards
- Pi-compatible power sources
- Pi-compatible USB wifi dongle
- Ethernet cable
- Private network with wifi and ethernet port
- VNC client on a wired or wireless device

The server doesn't require the onboard wifi of the Pi 3 so a Pi 2 could be substituted.

## Setup

### 1. Download the images and copy to sd cards

Copying the images to the cards is the same as for a regular Raspbian install. However to preserve the image size so that backup copies can safely be made, use the following Linux commands or their equivalents:

```
dd bs=1M count=7217 if=(server|client).img of=/dev/sdx
```
```
dd bs=1M count=7217 if=/dev/sdx of=backup.img
```

where sdx is the device name assigned to the card.

<br><br>

### 2. Insert the sd cards and power up the Pi's

<br><br>

### 3. Configure the server - Part I

The server is configured in two passes because of the need to exchange WireGuard keys between the Pi's.

#### 3.1 Connect the server to your network with an ethernet cable.

Modify local router settings to forward port 51820. You will probably want to either reserve a particular address for the server in the router's DHCP settings or assign a static address.

#### 3.2 Access the server using VNC

Determine the IP address assigned by your router to the server ethernet interface. In a VNC viewer app on a device on the local network connect to the server address using the default password, which is "raspberry". 

Start a terminal window on the VNC desktop.

#### 3.3 Update WireGuard

This could be done later after getting the current version working.

```
cd ~/WireGuard
sudo cp /etc/wireguard/wg0.conf /tmp
wget https://git.zx2c4.com/WireGuard/snapshot/WireGuard-x.x.x.tar.xz
tar xvf WireGuard-x.x.x.tar.xz

cd WireGuard-x.x.x/src
make
sudo make install

sudo cp /tmp/wg0.conf /etc/wireguard
```

#### 3.4 Configure WireGuard

```
cd ~/WireGuard
wg genkey | tee server_private_key | wg pubkey > server_public_key
sudo nano /etc/wireguard/wg0.conf
```

In /etc/wireguard/wg0.conf substitute the string from server_private_key into the file.

Copy the string from server_public_key over to your VNC viewer machine.

#### 3.5 Close the VNC connection

<br><br>

### 4. Configure the client Pi

#### 4.1 Access the client Pi using VNC

Disconnect the VNC viewer device from your local network and connect it to one of the private interfaces:

- wifi using the default SSID and password, "raspi" and "raspberry"
- ethernet cable

Then in the VNC viewer connect to the Pi address that corresponds to the interface:

- 10.100.100.1 for wifi
- 10.150.150.1 for ethernet

using the default password, which is "raspberry". 

Start a terminal window on the VNC desktop.

#### 4.2 Connect the public wifi interface

Use the wifi menu on the virtual desktop to connect the Pi to your local wifi network

#### 4.3 Update WireGuard

This could be done later after getting the current version working.

```
cd ~/WireGuard
sudo cp /etc/wireguard/wg0.conf /tmp
wget https://git.zx2c4.com/WireGuard/snapshot/WireGuard-x.x.x.tar.xz
tar xvf WireGuard-x.x.x.tar.xz

cd WireGuard-x.x.x/src
make
sudo make install

sudo cp /tmp/wg0.conf /etc/wireguard
```

#### 4.4 Configure WireGuard

```
cd ~/WireGuard
wg genkey | tee client_private_key | wg pubkey > client_public_key
sudo nano /etc/wireguard/wg0.conf
```

In /etc/wireguard/wg0.conf substitute the string from client_private_key into the file, then substiute the server_public_key value copied earlier from the server. Lastly substitute the public IP address of your network into the file. You can get this from your router user interface or by browsing a website like whoer.net. If the network IP is not very static then a DDNS hostname can also be used.

Copy the string from client_public_key over to your VNC viewer machine.

#### 4.5 Start WireGuard

```
sudo wg-quick up wg0
sudo systemctl enable wg-quick@wg0.service
```

Check that the "ifconfig" command shows a "wg0" interface has been created. From now on wireGuard will start on boot.

#### 4.6 Change default login passwords

- Change the pi user password with the passwd command
- Change the VNC password using the VNC settings menu on the virtual desktop

The Pi is also protected by a firewall on the public interface and the fact that the private wifi key is secret.

#### 4.7 Change default SSID and password of private wifi interface

The current values are 'raspi' and 'raspberry'. Edit the config file and restart hostapd.

```
sudo nano /etc/hostapd/hostapd.conf
sudo systemctl restart hostapd
```

Test the new passwords by reconnecting using the private wifi interface. If there are problems you can get back in using the wired connection to check settings or reboot.

#### 4.8 Close the VNC connection

<br><br>

### 5. Configure the server - Part II

#### 5.1 Reconnect to the server with VNC as before 

#### 5.2 Finish configuring WireGuard

```
sudo nano /etc/wireguard/wg0.conf
```

In /etc/wireguard/wg0.conf substitute the client_public_key value copied earlier from the client

#### 5.3 Start WireGuard

```
sudo wg-quick up wg0
sudo systemctl enable wg-quick@wg0.service
```

Check that the "ifconfig" command shows a "wg0" interface created. From now on WireGuard will start on boot.

#### 5.4 Define local DNS data - optional

Append local hostname/address data to /etc/unbound/unbound.conf.
It could be as simple as:

```
local-data: "myserver A 10.0.0.5"
```

Then restart unbound:

```
sudo systemctl restart unbound
```

#### 5.5 Change default login passwords

- Change the pi user password with the passwd command
- Change the VNC password using the VNC settings menu on the virtual desktop

Good to do even though it's behind a router.

#### 5.6 Close the VNC connection

<br><br>

## Test the VPN tunnel locally

Connect a device to one of the client private interfaces and start a browser on it.  If web access is working then you are almost finished (otherwise see the troubleshooting checks below). As a sanity check, confirm that there is traffic on the server's wg0 interface. This can be done on the server Pi desktop with the command "ifconfig wg0", which shows transmitted/received counts. Alternatively you can continuouly monitor traffic using "vnstat -l -i wg0". Lastly check that DNS traffic is not leaking outside the tunnel by accessing a website like dnsleak.com from your attached device browser.

<br><br>

## Test the VPN client remotely

Now it should be possible to connect to the server from any wifi location. A portable device with a VNC viewer app is required to start the connection. The device can connect using either the wifi or ethernet private network. The wifi widget on the VNC desktop is used to select the ssid and sign in. Captive portals will require starting a browser on the desktop to get access.

In a web browser on the attached device use a site like whoer.net to check that your ip address is the same as your WireGuard server. 

If the wifi interface is not needed it can be disabled/enabled with the commands:

````
sudo systemctl stop hostapd.service
sudo systemctl start hostapd.service
````

<br><br>

## Troubleshooting

- Check that both WireGuard interfaces are up as shown by ifconfig

- Check that wg0.conf files contain correct public/private keys

- Check that server ip address is correct in client's wg0.conf file

- Check that port 51820 is forwarded to the server

- Try a numeric ip web address to see if problem is with DNS. DNS lookups are handled by the unbound resolver running on the server.

- Try a browser on the client VNC desktop to confirm it has internet access.

- Check if server can be pinged through the tunnel from client VNC desktop - ping 10.200.200.1 If not then there is likely a problem with WireGuard configuration or routing has been modified. 

- Try a browser on the server VNC desktop to confirm it has internet access

<br><br>

## Technical Notes

- The wifi channel used by the client Pi's private interface is set in /etc/hostapd/hostapd.conf. It can be changed if needed to avoid interference. A reboot is needed following a change.
    
- In the client Pi setup for Raspbian Stretch (but not Jessie) there seems to be an incompatibility with using dhcpcd, dnsmasq, and hostapd to handle the interfaces and services that are needed. It's also has been noted in some forum posts. To work around this problem /etc/rc.local is used to start dhcpcd on boot because it is prevented from starting normally when /etc/network/interfaces contains interface definitions. This is a hack that may need to be revisited with future Raspbian releases.

## References

1. https://www.wireguard.com
2. https://twit.tv/shows/floss-weekly/episodes/468
3. [WireGuard VPN: Typical Setup](https://www.ckn.io)
4. [WireGuard VPN: Portable Raspberry Pi Setup](https://www.ckn.io)
5. [Building an encrypted travel wifi router](https://danrl.com)

