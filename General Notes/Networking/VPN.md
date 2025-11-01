
Virtual Private Network
- Operate at Layer 2 (Ethernet/Data Link layer)
	- Encapsulates entire Ethernet frames
- Some at Layer 3 (IP layer)
	- Encapsulates IP packets inside encrypted VPN packets
- Creates an encrypted connection over a public network to a privae network
	- Say I want to connect to company X's VPN
		- Initiation: establish test and get authenticated by VPN server
		- Tunnel establishment: VPN protocol (IPSec, OpenVPN, WireGuard) negotiates encryption and authentication keys. Once keys exchanged, VPN tunnel created
		- IP and routing changes: VPN assigns you an internal company IP. computer updates routing table so traffic to company resources goes through the VPN

What does "going through VPN (tunnel)" mean?
- Normally, just have source IP and destination IP, and packet goes directly through ISP into internet
	- Anyone on local network, my ISP, or any intermediate router can see source and destinatinon IP, even if payload is encrypted
- Now I use a VPN
	- Instead of sending packet directly to the target, the VPn wraps IP packet inside a new packet
	- The original packet is encrypted inside the payload, and **new destination IP is set to the VPN server IP**
	- So anyone observing the traffic only sees the encrypted data going to the VPN server and not the final website.
		- Don't even get the actual target IP, only the VPN IP
	- The VPN server receives the packet, decrypts the payload and extracts the original IP packet
		- Then forwards the packet to the destination on the internet
		- So to outsiders, the requests appears to come from the VPN server's IP, not my IP
	- The reverse happens for responses from target to me.

**Essentially, the VPN acts as a middleman, thus ie. an encrypted tunnel**.


me ----(encrypted data)---> VPN server ------------> website


# Connecting to a private network through VPN

Eg. I want to connect to a company's internal network through a VPN

- Similar as above explanation. 
- Additionally,
	- VPN server gives devices a **company-internal IP address**
	- Device updates routing table so traffic to company resources goes through the VPN tunnel
- From PC pov, it's on the internal network
- From internal network pov, your VPN IP is just another device on the network

ie.
```c
[Outer Ethernet Header]
    Src MAC: your NIC
    Dst MAC: router/gateway

[Outer IP Header]
    Src IP: your real public IP (203.0.113.42)
    Dst IP: VPN server IP (198.51.100.5)

[Transport Header]
    UDP/TCP depending on VPN protocol (e.g., UDP 51820 for WireGuard)

[VPN Payload]
    ENCRYPTED(
        [Inner IP Header]
            Src IP: VPN-assigned IP (10.50.0.23)
            Dst IP: internal company server IP (10.50.5.15)
        [Transport Header]
            TCP/UDP
        [Payload]
            HTTP request or app data
    )

```

- This is the entire VPN packet sent by PC to the VPN server
	- The VPN acts as if it sits inside the company's network, and r**esources sent from within network to PC are sent to the VPN-assigned IP**, then forwarded to PC ip.
	- So essentially the VPN maps (VPN-assigned IP (that it holds) -> PC IP), receives resources sent to VPN-assigned IP, then forwards to PC IP
- The VPN acts as a gateway for VPN clients

# Connecting to a VPN in linux
- `sudo openvpn user.ovpn`
	- openvpn is the client
	- user.ovpn is the VPN key
- If we then run ifconfig
	- should see a tun adapter if we successfully connected to the vpn
 ```c
$ ifconfig
tun0: flags=...  mtu 1500
       inet 10.10.14.5  netmask 255.255.255.0
       ...
```
- inet 10.10.14.5 - this is the VPN assigned IP inside the network
	- `netstat -rn` will show networks accessible via VPN


What is a tun adapter?
- This is the **virtual network interface** created by the OS
	- ie a virtual Network Interface Card
- OS can send packets to this interface just like a real Ethernet/Wifi-interface (eg. eth0, wlan0)
- OpenVPN captures packets from the TUN interface, encrypts them, then sends them through the real network to the VPn server
- Packets coming back from the VPN server decrypted by OpenVPN and injected into the TUN interface so that OS thinks they came from the VPN-assigned network

# Routing and interfaces
- OS uses a routing table to decide which interface to send packets through, based on destination IP address
- eg. linux -> ip route
```c
default via 192.168.1.1 dev wlan0
10.10.14.0/24 dev tun0
192.168.1.0/24 dev wlan0
```
dev wlan0 -> traffic goes out Wifi interface
dev tun0 -> traffic goes through VPN tunnel

- When I connect to a VPN,
	- OpenVPN adds routes automatically to the routing table
	- So Traffic destined for the VPN network (10.10.14.0/24) / even all internet traffic (0.0.0.0/0) is sent through tun0

Manually choose which interface to route through:
- Specify interface in a command
	- eg. ping -I tun0 10.10.14.5
- Modify routing table
	- ip route add 8.8.8.832 dev wlan0
		- -> Send traffic 8.8.8.8 via wi-fi, even if vpn is default route