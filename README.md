# HAProxy in pfSense as a Reverse Proxy
If you want convenient remote access to your LXC's and/or Apps the easiest way is to setup HAProxy application addon in pfSense. Then you will have access to your Apps using address URLS like:
> *  unifi.myserver.com --> unifi 192.168.1.251
> * appname.myserver.com --> appname 192.168.1.XXX

pfSense package manager has a ready built distribution of HAProxy.

This guide helped me so much and much of what I have written here was harvested from this [GUIDE](https://www.chucknemeth.com/pfsense-haproxy-port-443/)

Network prerequisites are:
- [x] Layer 2 Network Switches
- [x] Network Gateway is `192.168.1.5`
- [x] Network DNS server is `192.168.1.5` (Note: your Gateway hardware should enable you to a configure DNS server(s), like a UniFi USG Gateway, so set the following: primary DNS `192.168.1.254` which will be your PiHole server IP address; and, secondary DNS `1.1.1.1` which is a backup Cloudfare DNS server in the event your PiHole server 192.168.1.254 fails or os down)
- [x] Network DHCP server is `192.168.1.5`

Other Prerequisites are:
- [x] Proxmox node fully configured as per [PROXMOX-NODE BUILDING](https://github.com/ahuacate/proxmox-node/blob/master/README.md#proxmox-node-building)
- [x] pfSense is fully configured on typhoon-01 including both OpenVPN Gateways VPNGATE-LOCAL and VPNGATE-WORLD.
- [x] You own a registered Domain name

Tasks to be performed are:
- [ ] 1.0 Proxmox Base OS Installation
- [ ] 2.0 Prepare your Network Hardware - Ready for Typhoon-01
- [ ] 3.0 Easy Installation Option
- [ ] 4.0 Basic Proxmox node configuration
- [ ] 5.0 Create a Proxmox pfSense VM on typhoon-01

## 1.0 Create a Cloudfare Acccount
My domain management is done by Google Domains only because it was the cheapest provider at the time (migrated from GoDaddy). But I have configured my domains DNS name servers to Cloudfare servers.

So this tuturial will refer to Cloudfare DNS management.

I recommend you too setup and migrate your domain DNS management to Cloudfare. Not only are there servers fast, they also have a API to configure your DNS records and a free Dynamic DNS service accessed by a API key. If you too want to use Cloudfare DNS name servers you can create a free account at Cloudfare. There are plenty of tutorials about how to move your DNS services from Google, Godaddy and others to Cloudfare on the internet.

### 1.1 Creating an Cloudfare DNS A record for your home server(s)
First you must decide on your subdomain names. It’s part of the address used to direct traffic to a particular service running on your servers. For example, **jellyfin.`bahamas`.myserver.com** or **jellyfin.`zurich`.myserver.com** where **`bahamas`** and **`zurich`** are two different locations in the world. 



