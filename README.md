# HAProxy in pfSense as a Reverse Proxy
If you want convenient remote access to your LXC's and/or Apps the easiest way is to setup HAProxy application addon in pfSense. Then you will have access to your Apps using address URLS like:
> *  unifi.myserver.com --> unifi 192.168.1.251
> * appname.myserver.com --> appname 192.168.1.XXX

pfSense package manager has a ready built distribution of HAProxy.

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

## 1.0 Create a Dynamic DNS Service
Here I recommend using a free Dynamic DNS Service like https://freedns.afraid.org . Setup a free account. Then go to the section `For Members` > `Dynamic DNS` > `Add` and configure the `Add a new subdomain` template as follows:

| Add a new subdomain | Value | Notes
| :---  | :---: | :---
| Type | `A` |
| Subdomain| Enter a subdomain ID of your choosing | *whatever you type here, such as `kingkong` will create a address URL `kingkong.crabdance.com`. Choose something you remember.*
| Domain | `crabdance.com (public)` | *Select crabdance.com*
| Destination | Leave blank
| TTL | Leave blank
| Wildcard | ☐ | *Uncheck*

And complete the verification request and click `Save`.

Next we need to obtain your new DDNS subdomain service provider access token. To get your access token go to the section `For Members` > `Dynamic DNS` and you should see your newly created subdomain. Hover on the hyperlink `Direct URL` and copy the link location/target of `Direct URL`. In the URL you copied the code after `?` is your `Access Token`, freedns.afraid.org/dynamic/update.php?YOUR_UPDATE_TOKEN_IS_HERE. 



