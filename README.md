# WireGuard Portable VPN Client and Server on Raspberry Pi 3's

![alt text](https://github.com/jimb245/2PiWireGuardVPN/blob/master/2PiVPN.jpg)

This post describes how you can set up and use a portable WireGuard VPN access point and server on Raspberry Pi 3's starting from the provided Raspbian-based image files. The images already contain all the necessary software so setup consists of installing the images on the Pi's and editing some parameters like passwords and IP addresses. For general information about WireGuard see References 1-2 below. References 3-5 are install guides for some WireGuard configurations on Ubuntu and Raspbian. The images provided here are adapted from those guides.

To use this 2-Pi VPN you connect the server with an ethernet cable to a protected network that has internet access. UDP packets from the internet must be able to reach the WireGuard port on the server. For example, it could be connected to an unused jack on a home router with port forwarding configured. 

The portable access point is similar to a travel router, with both public and private interfaces. You connect the public interface to a public wifi and connect your portable devices to the private interface, using either wifi or an ethernet cable. In practice you first connect to the private interface with a portable device that has a VNC client. Using the VNC desktop you connect to public wifi and if necessary start a browser in order to handle captive portals. Lastly you start the WireGuard connection - it does not start on boot because it would interfere with accessing captive portal servers. At the end of the session you use VNC again to safely shut down the Pi.

If you are connecting your portable device over ethernet then the private wifi interface is not needed and optionally you can shut it down to avoid interference or to save power when using a battery with the Pi. Portable devices that lack an ethernet port may still be usable in this mode with an adapter if they support USB OTG.

Of course installing the server on your home network won't help if your goal is to prevent snooping by your home ISP. For that you could consider using a cloud server. Reference 3. below explains how to set up a WireGuard server on a Ubuntu system, for example. The portable access point could be used with either server. One thing to note is that some video streaming services will block traffic that appears to come from the well-known cloud providers so you may not want to funnel your entire home network through a cloud VPN.

Software versions on the images:
- Raspbian Stretch (March 2018)
- WireGuard 0.0.20180304

## What You Will Need

- Two Raspberry Pi 3's with cases
- Two 8GB microSD cards
- Pi-compatible power supplies
- Pi-compatiblle battery and cable - optional
- Pi-compatible USB wifi dongle
- Ethernet cable
- Private network with wifi and ethernet port
- VNC client on a wired or wireless device

The server doesn't require the onboard wifi of the Pi 3 so a Pi 2 could be substituted.

## Setup

### 1. Download the images and copy to sd cards

Copying the images to the cards is the same as for a regular Raspbian install.

<br><br>

### 2. Insert the sd cards and power up the Pi's

<br><br>

### 3. Configure the server - Part I

The server is configured in two passes because of the need to exchange WireGuard keys between the Pi's.

#### 3.1 Connect the server to your network with an ethernet cable.

#### 3.2 Access the server using VNC

Determine the IP address assigned by your router to the server ethernet interface. You can usually find this by logging into your router's user interface. Alternatively phone apps like Fing can list devices on your network. In your VNC client connect to the server address with the default password, which is "raspberry". Start a terminal window on the VNC desktop.

#### 3.3 Configure WireGuard

```cd ~/WireGuard
Umask 077
wg genkey | tee server_private_key | wg pubkey > server_public_key
sudo nano /etc/wireguard/wg0.conf
```

In /etc/wireguard/wg0.conf substitute the string from server_private_key into the file.

Copy the string from server_public_key over to your VNC client machine.

#### 3.4 Close the VNC connection

<br><br>

### 4. Configure the client

#### 4.1 Access the client Pi using VNC

Disconnect the VNC device from your main network and connect it to one of the private interfaces:

- wifi using the default SSID and password, "raspi" and "raspberry"
- ethernet cable

Then connect to VNC with the default password, which is "raspberry". 

#### 4.2 Connect the public wifi interface

Use the wifi menu on the virtual desktop to connect the Pi to your wifi network

#### 4.3 Configure WireGuard

Start a terminal window on the VNC desktop.

```cd ~/WireGuard
Umask 077
wg genkey | tee client_private_key | wg pubkey > client_public_key
sudo nano /etc/wireguard/wg0.conf
```

In /etc/wireguard/wg0.conf substitute the string from client_private_key into the file, then substiute the server_public_key value copied earlier from the server. Lastly substitute the public IP address of your network into the file. You can get this from your router user interface or by browsing a website like whoer.net.

Copy the string from client_public_key over to your VNC client machine.

#### 4.4 Start WireGuard

sudo wg-quick up wg0

#### 4.5 Change default login passwords

- Change the pi user password with the passwd command
- Change the VNC password using the VNC settings menu on the virtual desktop

#### 4.6 Change default SSID and password of private wifi interface

```sudo nano /etc/hostapd/hostapd.conf
sudo systemctl restart hostapd
```

Test the new passwords by reconnecting using the private wifi interface.

#### 4.7 Close the VNC connection

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
```

#### 5.4 Change default login passwords

- Change the pi user password with the passwd command
- Change the VNC password using the VNC settings menu on the virtual desktop

#### 5.5 Close the VNC connection

<br><br>

## Test the VPN tunnel locally

Connect a device to one of the client private interfaces and start a browser.  If web access is working then you are almost finished. As a sanity check, confirm that there is traffic on the server's wg0 interface. This can be done on the server desktop with the command "ifconfig wg0", which shows transmitted/received counts. Alternatively you can continuouly monitor traffic using "vnstat -l -i wg0". Lastly check that DNS traffic is not leaking outside the tunnel by accessing a website like dnsleak.com from your device browser.

<br><br>

## Test the VPN client remotely

Now you should be able to connect to the server from any wifi location. Remember to start WireGuard after establishing the wifi connection on the VNC desktop:

```
sudo wg-quick up wg0
```

Use a site like whoer.net to check that your ip address is the same as your WireGuard server. If the wifi interface is not needed it can be disabled/enabled with the commands:

````
sudo ifdown wlan1
sudo ifup wlan1
````

<br><br>

## Troubleshooting

- Check that both WireGuard interfaces are up as shown by ifconfig

- Check that wg0.conf files contain correct public/private keys

- Check that server ip address is correct in client's wg0.conf file

- Check that port 51820 is forwarded to the server

- Try a numeric ip web address to see if problem is with DNS. DNS lookups are handled by the unbound resolver running on the server.

- Try a browser on the client VNC desktop to see if problem is between your device and the Pi

- Check if server can be pinged through the tunnel from client VNC desktop - ping 10.200.200.1 If not then there is likely a problem with WireGuard configuration or routing has been modified. 

- Try a browser on the server VNC desktop to confirm it has internet access

<br><br>

## Technical Notes

- The wifi channel used by the client Pi's private interface is set in /etc/hostapd/hostapd.conf. It can be changed if needed to avoid interference. A reboot is needed following a change.

- It would be possible to have a Pi 3 server connect to the local network over wifi rather than ethernet, though in general the ethernet connection likely will provide better performance. To do this add the following iptables entry. The initial wifi authentication will require accessing the server another way, which could be a temporary ethernet connection or attaching keyboard and monitor to the Pi. 

```
    sudo iptables -t nat -A POSTROUTING -s 10.200.200.0/24 -o wlan0 -j MASQUERADE
    netfilter-persistent save
```
    
- In the client Pi setup for Raspbian Stretch (but not Jessie) there seems to be a basic conflict when combining dhcpcd, dnsmasq, and hostapd services to provide both a DHCP-generated ip addess for the public interface and a static ip for the private wifi interface. To work around this problem /etc/rc.local is used to start dhcpcd on boot because it is prevented from starting normally. This is a hack that may need to be revisited in the future.

## References

1. https://www.wireguard.com
2. https://twit.tv/shows/floss-weekly/episodes/468
3. https://www.ckn.io/blog/2017/11/14/wireguard-VPN-typical-setup.html
4. https://www.ckn.io/blog/2017/12/28/wireguard-VPN-portable-raspberry-pi-setup.html
5. https://danrl.com/blog/2016/travel-wifi.html

