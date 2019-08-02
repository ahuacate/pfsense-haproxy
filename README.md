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
My domain management is done by Google Domains. But I have redirected my domains DNS Name Server with another provider called Cloudfare.

I recommend you too redirect your domain DNS Names Servers to Cloudfare. Not only are there servers fast, they also have an API Key to configure your DNS records automatically and provide a free Dynamic DNS service. If you too want to use Cloudfare DNS name servers you can create a free account at Cloudfare. There are plenty of tutorials about how to move your DNS services from Google, Godaddy and other providers to Cloudfare on the internet.

So this tuturial will refer to Cloudfare DNS management from now on.

## 2.0 Configure your domains at Cloudfare
First you must decide on your subdomain names. It’s part of the address used to direct traffic to a particular service running on your servers. For example, **jellyfin.`site1`.myserver.com** or **jellyfin.`site2`.myserver.com** where **`site1`** and **`site2`** are two different locations (i.e cities) in the world.

### 2.1 Create DNS A records for your servers
First login to your Cloudfare Dashboard Home, choose your domain and go to `DNS TAB`. You will be provided with a page to `Manage your Domain NAme System (DNS) settings`. Using the Cloudfare web interface create the following form entries by clicking `Add Record` after completing each each entry:

| Type | Name | IPv4 address | Automatic TTL | Orange Cloud | Notes
| :---: | :---: | :---: | :---: | :---: | :---
| `A` | `jellyfin.location1` | 0.0.0.0 | `Automatic TTL` | `OFF` | *Note, Uncheck the cloudfare orange cloud.*

### 2.2 Disable Cloudfare Crypto
Using your Cloudfare Dashboard Home, choose your domain and go to `Crypto TAB`. Under the section `SSL - Encrypt communication to and from your website using SSL` disable the service by setting it to the `Off` state.

## 3.0 pfSense Dynamic DNS
Cloudfare provides you with a API key (called the Global API Key)which gives pfSense the rights to update your domains DNS information. So have your Cloudfare Global API key ready by:
*  Log in to Cloudflare Account and go to your Profile;
*  Scroll down and View your **Global API Key**;
*  Complete the password challenge and note your key.

### 3.1 Create pfSense Dynamic DNS entries
We need to configure pfSense to send the DynamicDNS infornmation to Cloudflare. In the pfSense WebGUI go to `Services` > `Dynamic DNS` > `Dynamic DNS Clients`. Click `Add` and fill out the necessary fields as follows:

| Dynamic DNS Client | Value | Notes
| :--- | :--- | :---
| **jellyfin.location1.myserver.com**
| Disable | `☐` Disable this client |*Uncheck*
| Service Type | `Cloudfare`
| Interface to monitor | `WAN`
| Hostname | `jellyfin.location1`
| Domain | `myserver.com` | *Replace with your domain name.*
| MX | Leave Blank
| Wildcards | `☑` Enable Wildcard
| Cloudflare Proxy | Enable Proxy
| Verbose logging | `☑` Enable verbose logging
| Username | Enter your Cloudfare Accounts reg'd Email Address
| Password | Enter your Global API Key | *See section 3.0*
| Description | `jellyfin.location1.myserver.com`




