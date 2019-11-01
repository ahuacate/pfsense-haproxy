# HAProxy in pfSense
A reverse proxy server is a type of proxy server that typically sits behind a firewall in a private network and directs client requests to the appropriate backend server. A reverse proxy provides an additional level of abstraction and control to ensure the smooth flow of network traffic between clients and servers.

If you want a convenient remote internet access to your LXC's and/or Apps within your network the easiest way is to setup HAProxy which is addon in pfSense.

With HAProxy you will have access to your Apps and internal servers using address URLS like:
>  https://unifi-site1.foo.bar --> unifi 192.168.1.251

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
- [ ] 1.0 Create a Cloudfare Acccount
- [ ] 2.0 Configure your domains at Cloudfare
- [ ] 3.0 pfSense Dynamic DNS
- [ ] 4.0 Install ACME on pfSense
- [ ] 5.0 Generate ACME Certificates
- [ ] 6.0 Install HAProxy
- [ ] 7.0 Edit your UniFi network firewall
- [ ] 8.0 HAProxy Settings
- [ ] 9.0 HAProxy Frontend Settings
- [ ] 10.0 HAProxy Backend Settings
- [ ] 11.0 Fix for pfSense Dynamic DNS

## 1.0 Create a Cloudfare Acccount
I recommend you redirect your domain DNS Names Servers to Cloudfare. Not only are Cloudfare DNS servers fast, they also have an API Key for configuring your DNS records automatically and provide a **free Dynamic DNS service**. If you want to use Cloudfare DNS name servers you can create a free account at Cloudfare.

There are plenty of tutorials about how to move your DNS services from Google, Godaddy and other providers to Cloudfare on the internet.

This tutorial will refer to Cloudfare DNS management from now on.

## 2.0 Configure your domains at Cloudfare
First you must decide on your subdomain names. It’s part of the address used to direct traffic to a particular service running on your sites servers. 

For example, **jellyfin-`site1`.foo.bar** or **jellyfin-`site2`.foo.bar** where **`site1`** and **`site2`** are two different locations (i.e cities) in the world.

### 2.1 Create DNS A records for your servers
First login to your Cloudfare Dashboard Home, choose your domain and go to `DNS TAB`. You will be provided with a page to `Manage your Domain NAme System (DNS) settings`. Using the Cloudfare web interface create the following form entries by clicking `Add Record` after completing each each entry:

| Type | Name | IPv4 address | Automatic TTL | Orange Cloud | Notes
| :---: | :---: | :---: | :---: | :---: | :---
| `A` | `jellyfin-site1` | 0.0.0.0 | `Automatic TTL` | `OFF` | *Note, Uncheck the cloudfare orange cloud. Also the IP address 0.0.0.0 will be updated by your pfSense DDNS service.*
| `A` | `radarr-site1` | 0.0.0.0 | `Automatic TTL` | `OFF` |
| `A` | `sonarr-site1` | 0.0.0.0 | `Automatic TTL` | `OFF` |
| `A` | `nzbget-site1` | 0.0.0.0 | `Automatic TTL` | `OFF` |
| `A` | `deluge-site1` | 0.0.0.0 | `Automatic TTL` | `OFF` |
| `A` | `ombi-site1` | 0.0.0.0 | `Automatic TTL` | `OFF` |
| `A` | `syncthing-site1` | 0.0.0.0 | `Automatic TTL` | `OFF` |
| `A` | `vpn-site1` | 0.0.0.0 | `Automatic TTL` | `OFF` |

### 2.2 Cloudfare Crypto
Using your Cloudfare Dashboard Home, choose your domain and go to `Crypto TAB`. Using the Cloudfare web interface edit the following form entries to match the table below:

| Crpto | Value | Notes
| :---: | :---: | :---
| SSL | `Full` | *The section should show **Universal SSL Status Active Certificate***
| Edge Certificates | Leave Default
| Custom Hostnames | Leave Default
| Origin Certificates | Leave Default
| Always Use HTTPS | `On`
| HTTP Strict Transport Security (HSTS) | Leave Default
| Authenticated Origin Pulls | `On`
| Minimum TLS Version | `TLS 1.0 (default)`
| Opportunistic Encryption | `On`
| Onion Routing  | `On`
| TLS 1.3  | `On`
| Automatic HTTPS Rewrites | `Off`
| Disable Universal SSL | Leave Default

## 3.0 pfSense Dynamic DNS
Cloudfare provides you with a API key (called the Global API Key)which gives pfSense the rights to update your domains DNS information. So have your Cloudfare Global API key ready by:
*  Log in to Cloudflare Account and go to your Profile;
*  Scroll down and View your **Global API Key**;
*  Complete the password challenge and note your key.

### 3.1 Create pfSense Dynamic DNS entries
We need to configure pfSense to send the DynamicDNS infornmation to Cloudflare. In the pfSense WebGUI go to `Services` > `Dynamic DNS` > `Dynamic DNS Clients`. Click `Add` and fill out the necessary fields as follows:

| Dynamic DNS Client | Value | Notes
| :--- | :--- | :---
| **jellyfin-site1.foo.bar**
| Disable | `☐` Disable this client |*Uncheck*
| Service Type | `Cloudfare`
| Interface to monitor | `LAN`
| Hostname | `jellyfin-site1` | *Replace site1 with your location i.e home, mancave, beachhouse*
| Domain | `foo.bar` | *Replace with your domain name.*
| MX | Leave Blank
| Wildcards | `☐ Enable Wildcard` | *Disable wildcards*
| Cloudflare Proxy | `☑ Enable Proxy`
| Verbose logging | `☑` Enable verbose logging
| Username | Enter your Cloudfare Accounts reg'd Email Address
| Password | Enter your Global API Key | *See section 3.0*
| Description | `jellyfin-site1.foo.bar`

And click `Save & Force Update`. Now repeat the above steps for all your Cloudfare DNS A-record entries and servers (i.e `radarr-site1`, `sonarr-site1`, `nzbget-site1`, `deluge-site1`, `vpn-site1` etc)

Then check your Cloudfare DNS A-records your created [HERE](https://github.com/ahuacate/proxmox-reverseproxy/blob/master/README.md#21-create-dns-a-records-for-your-servers) and all your servers IP values should change from 0.0.0.0 to your WAN IP address.

## 4.0 Install ACME on pfSense
We need to install the ACME package on your pfSense. ACME is Automated Certificate Management Environment, for automated use of LetsEncrypt certificates.

In the pfSense WebGUI go to `System` > `Package Manager` > `Available Packages Tab` and search for `ACME`. Install the `ACME` package.

### 4.1 ACME General Settings
In the pfSense WebGUI go to `Service` > `ACME` > `Settings` and fill out the necessary fields as follows:

| General Settings Tab | Value 
| :--- | :---
| Cron Entry | `☑ Enable Acme client renewal job` 
| Write Certificates | Leave Blank

## 5.0 Generate ACME Certificates
We will need to generate certificates from a trusted provider such as Let’s Encrypt.

We can use the ACME Package provided in pfSense.

### 5.1 Create ACME Account Keys
First you need to create some account keys. LetsEncrypt is rate limited so you want to make sure that you have everything configured correctly before requesting a real cert. To help people test, LetsEncrypt provides a test service that you can use as you figure out your settings without bumping into the rate limit on the production servers. Certs obtained from these test services cannot be used.

So we will create two Account Keys.

In the pfSense WebGUI go to `Services` > `Acme Certificates` > `Account Keys`. Click `Add` and fill out the necessary fields as follows:

| Edit Certificate options | Value | Notes
| :--- | :--- | :---
| **Production Key**
| Name | `site1.foo-production` | *For example, `site1.foo-production`*
| Description | `site1.foo-production key` | *For example, `site1.foo-production key`*
| Acme Server | `Let’s Encrypt Production ACME v2 (Applies rate limits to certificate requests)`
| E-Mail Address | Enter your email address
| Account key | `Create new account key` | *Click `Create new account key`*
| Acme account registration | `Register acme account key` | *Click `Register acme account key`*
| **Test Key**
| Name | `site1.foo-test` | *For example, `site1.foo-test`*
| Description | `site1.foo-test key` | *For example, `site1.foo-test key`*
| Acme Server | `Let’s Encrypt Staging ACME v2 (for TESTING purposes)`
| E-Mail Address | Enter your email address
| Account key | `Create new account key` | *Click `Create new account key`*
| Acme account registration | `Register acme account key` | *Click `Register acme account key`*

It is really important that you choose the Staging ACME v2 server. Only the v2 will support wildcard domains.

Now click the `+ Create new account key` button and wait for the box to fill in with a new RSA private key.

Then click the `Register ACME Account key`. The little cog will spin and if it worked the cog will turn into a check. 

Finally click `Save`.

### 5.2 Create ACME Certificates
In the pfSense WebGUI go to `Services` > `Acme Certificates` > `Certificates`. Click `Add` and fill out the necessary fields as follows. Notice I have multiple entries in the Domain SAN List. This means the same certificate will be used for each server connection. In this example we will get a certificate that covers `jellyfin-site1`, `radarr-site1`, `sonarr-site1`, `nzbget-site1` and `deluge-site1` - basically all your media server connections.

| Edit Certificate options | Value
| :--- | :---
| Name | `media-site1.foo.bar`
| Description | `site1.foo.bar media SSL Certificate`
| Status | `active`
| Acme Account | `site1.foo-test`
| Private Key | `384-bit ECDSA`
| OCSP Must Staple | `☐ Add the OCSP Must Staple extension to the certificate`
| **Entry 1 - Domain SAN list**
| Mode |`Enabled`
| Domainname `jellyfin-site1.foo.bar` | *One entry per Cloudfare DNS A-record*
| Method | `DNS-Cloudflare`
| Mode | `Enabled`
| Key | Fill in the Cloudfare Global API Key
| Email | Enter your email address
| Enable DNS alias mode | Leave Blank
| Enable DNS domain alias mode | `☐ (Optional) Uses the challenge domain alias value as --domain-alias instead in the acme.sh call`
| **Entry 2 - Domain SAN list**
| Mode |`Enabled`
| Domainname `radarr-site1.foo.bar` | *One entry per Cloudfare DNS A-record*
| Method | `DNS-Cloudflare`
| Mode | `Enabled`
| Key | Fill in the Cloudfare Global API Key
| Email | Enter your email address
| Enable DNS alias mode | Leave Blank
| Enable DNS domain alias mode | `☐ (Optional) Uses the challenge domain alias value as --domain-alias instead in the acme.sh call`
| **Entry 3 - Domain SAN list**
| Mode |`Enabled`
| Domainname `sonarr-site1.foo.bar` | *One entry per Cloudfare DNS A-record*
| Method | `DNS-Cloudflare`
| Mode | `Enabled`
| Key | Fill in the Cloudfare Global API Key
| Email | Enter your email address
| Enable DNS alias mode | Leave Blank
| Enable DNS domain alias mode | `☐ (Optional) Uses the challenge domain alias value as --domain-alias instead in the acme.sh call`
| **Entry 4 - Domain SAN list**
| Mode |`Enabled`
| Domainname `nzbget-site1.foo.bar` | *One entry per Cloudfare DNS A-record*
| Method | `DNS-Cloudflare`
| Mode | `Enabled`
| Key | Fill in the Cloudfare Global API Key
| Email | Enter your email address
| Enable DNS alias mode | Leave Blank
| Enable DNS domain alias mode | `☐ (Optional) Uses the challenge domain alias value as --domain-alias instead in the acme.sh call`
| **Entry 5 - Domain SAN list**
| Mode |`Enabled`
| Domainname `deluge-site1.foo.bar` | *One entry per Cloudfare DNS A-record*
| Method | `DNS-Cloudflare`
| Mode | `Enabled`
| Key | Fill in the Cloudfare Global API Key
| Email | Enter your email address
| Enable DNS alias mode | Leave Blank
| Enable DNS domain alias mode | `☐ (Optional) Uses the challenge domain alias value as --domain-alias instead in the acme.sh call`
| **Entry 6 - Domain SAN list**
| Mode |`Enabled`
| Domainname `ombi-site1.foo.bar` | *One entry per Cloudfare DNS A-record*
| Method | `DNS-Cloudflare`
| Mode | `Enabled`
| Key | Fill in the Cloudfare Global API Key
| Email | Enter your email address
| Enable DNS alias mode | Leave Blank
| Enable DNS domain alias mode | `☐ (Optional) Uses the challenge domain alias value as --domain-alias instead in the acme.sh call`
| **Entry 7 - Domain SAN list**
| Mode |`Enabled`
| Domainname `syncthing-site1.foo.bar` | *One entry per Cloudfare DNS A-record*
| Method | `DNS-Cloudflare`
| Mode | `Enabled`
| Key | Fill in the Cloudfare Global API Key
| Email | Enter your email address
| Enable DNS alias mode | Leave Blank
| Enable DNS domain alias mode | `☐ (Optional) Uses the challenge domain alias value as --domain-alias instead in the acme.sh call`
| **Remaining fields**
| DNS Sleep | Leave Blank
| Action List `Add` | `/etc/rc.restart_webgui`
| Last renewal | Leave Blank
| Certificate renewal after | `60`

Then click `Save`  followed by `Issue/Renew`. A review of the output will appear on the page and if successful you see a RSA key has been generated. The output should begin like this:

```
media-site1.foo.bar
Renewing certificate
account: foo-test
server: letsencrypt-staging-2


/usr/local/pkg/acme/acme.sh --issue -d 'jellyfin-site1.foo.bar' --dns 'dns_cf' -d 'sonarr-site1.foo.bar' --dns 'dns_cf' -d 'radarr-site1.foo.bar' --dns 'dns_cf' -d 'nzbget-site1.foo.bar' --dns 'dns_cf' -d 'deluge-site1.foo.bar' --dns 'dns_cf' -d  --home '/tmp/acme/media-site1.foo.bar/' --accountconf '/tmp/acme/media-site1.foo.bar/accountconf.conf' --force --reloadCmd '/tmp/acme/media-site1.foo.bar/reloadcmd.sh' --log-level 3 --log '/tmp/acme/media-site1.foo.bar/acme_issuecert.log'

Array
(
[path] => /etc:/bin:/sbin:/usr/bin:/usr/sbin:/usr/local/bin/
[PATH] => /etc:/bin:/sbin:/usr/bin:/usr/sbin:/usr/local/bin/
[CF_Key] => 8XXXXXXXXXXXXXXXXXXXXXXXXXXXXX
[CF_Email] => yourname@example.com
)
[Sat Aug 3 14:34:55 +07 2019] Registering account
[Sat Aug 3 14:34:58 +07 2019] Already registered
[Sat Aug 3 14:34:58 +07 2019] ACCOUNT_THUMBPRINT='XXXXXXXXXXXXXXXXXXXXXXXXXXXXX'
[Sat Aug 3 14:34:58 +07 2019] Multi domain='DNS:jellyfin-site1.foo.bar,DNS:sonarr-site1.foo.bar,radarr-site1.foo.bar,nzbget-site1.foo.bar,deluge-site1.foo.bar'
[Sat Aug 3 14:34:58 +07 2019] Getting domain auth token for each domain
[Sat Aug 3 14:35:12 +07 2019] Getting webroot for domain='jellyfin-site1.foo.bar'
[Sat Aug 3 14:35:12 +07 2019] Getting webroot for domain='sonarr-site1.foo.bar'
[Sat Aug 3 14:35:12 +07 2019] Getting webroot for domain='radarr-site1.foo.bar'
[Sat Aug 3 14:35:12 +07 2019] Getting webroot for domain='nzbget-site1.foo.bar'
[Sat Aug 3 14:35:12 +07 2019] Getting webroot for domain='deluge-site1.foo.bar'
[Sat Aug 3 14:35:12 +07 2019] jellyfin-site1.foo.bar is already verified, skip dns-01.
[Sat Aug 3 14:35:12 +07 2019] sonarr-site1.foo.bar is already verified, skip dns-01.
[Sat Aug 3 14:35:12 +07 2019] radarr-site1.foo.bar is already verified, skip dns-01.
[Sat Aug 3 14:35:12 +07 2019] nzbget-site1.foo.bar is already verified, skip dns-01.
[Sat Aug 3 14:35:12 +07 2019] deluge-site1.foo.bar is already verified, skip dns-01.
[Sat Aug 3 14:35:12 +07 2019] Verify finished, start to sign.
[Sat Aug 3 14:35:12 +07 2019] Lets finalize the order, Le_OrderFinalize: https://acme-v02.api.letsencrypt.org/acme/finalize/XXXXXXXXXXXXXXXXXXXXX
[Sat Aug 3 14:35:15 +07 2019] Download cert, Le_LinkCert: https://acme-v02.api.letsencrypt.org/acme/cert/XXXXXXXXXXXXXXXX
[Sat Aug 3 14:35:18 +07 2019] Cert success
-----BEGIN CERTIFICATE-----
your key is here
-----END CERTIFICATE-----
```

Once you're satisfied everything is configured correctly, edit the certificate and change the Acme Account from `site1.foo-test` to  `site1.foo-production` and repeat the `Issue/Renew` steps to generate a usable certificate.

Final validation of your newly created LetsEncrypt certificate can be done by going to `System` > `Certificate Manager` > `Certificates`. It will show the issuer as something like **“Acmecert: 0=Let’s Encrypt,CN=Let’s Encrypt Authority X3,C=US”**.


## 6.0 Install HAProxy
We need to install the HAProxy package on your pfSense.

In the pfSense WebGUI go to `System` > `Package Manager` > `Available Packages Tab` and search for `HAProxy`. Install `haproxy` package.

## 7.0 Edit your UniFi network firewall
You should've already done this task if you followed these [instructions](https://github.com/ahuacate/proxmox-node/blob/master/README.md#25-edit-your-unifi-network-firewall). If not here they are again.

On your Proxmox Qotom build (typhoon-01) NIC ports enp3s0 & enp4s0 are bonded to create LAG `bond1`. You will then create in Proxmox a Linux Bridge using `bond1` called `vmbr2`. When you install pfSense VM on typhoon-01 the pfSense and HAProxy software will assign `vmbr2` (bond1) as its WAN interface NIC.

This WAN interface is VLAN2 and named in the UniFi controller software as `VPN-egress`. It's configured with network `Guest security policies` in the UniFi controller therefore it has no access to other network VLANs. The reason for this is explained build recipe for `VPN-egress` shown [HERE](https://github.com/ahuacate/proxmox-node#22-create-network-switch-vlans).

For HAProxy to work you must authorise VLAN2 (WAN in pfSense HAProxy) to have access to your Proxmox LXC server nodes with static IPv4 addresses on VLAN50.

The below instructions are for a UniFi controller `Settings` > `Guest Control`  and look under the `Access Control` section. Under `Pre-Authorization Access` click`**+** Add IPv4 Hostname or subnet` to add the following IPv4 addresses to authorise access for VLAN2 clients:fill out the form details as shown below:

| + Add IPv4 Hostname or subnet | Value | Notes
| :---  | :---: | :---
| IPv4 | 192.168.50.111 | *Jellyfin Server*
| IPv4 | 192.168.50.112 | *Sonarr Server*
| IPv4 | 192.168.50.113 | *Radarr Server*
| IPv4 | 192.168.50.114 | *Nzbget Server*
| IPv4 | 192.168.50.115 | *Deluge Server*

And click `Apply Changes`.

As you've probably concluded you must add any new HAProxy backend server IPv4 address(s) to the Unifi Pre-Authorization Access list for HAProxy frontend to have access to those backend VLAN50 servers.

## 8.0 HAProxy General Settings
In the pfSense WebGUI go to `Service` > `ACME` > `Settings` and fill out the necessary fields as follows:

| Settings Tab | Value 
| :--- | :--- 
| Enable HAProxy | `☑` 
| Maximum connections | `256`
| Number of processes to start | `1`
| Number of theads to start per process | `1`
| Reload behaviour | `☑ Force immediate stop of old process on reload. (closes existing connections)`
| Reload stop behaviour | Leave default
| Carp monitor | `Disabled`
| **Stats tab, 'internal' stats port**
| Internal stats port  | `2200`
| Internal stats refresh rate | Leave blank
| Sticktable page refresh rate | Leave blank
| **Logging**
| Remote syslog host | Leave blank
| Syslog facility | `local0`
| Syslog level | `Informational`
| Log hostname | Leave blank
| **Global DNS resolvers for haproxy**
| DNS servers | Leave blank
| Retries | Leave blank
| Retry timeout | Leave blank
| Interval | Leave blank
| **Global email notifications**
| Mailer servers | Leave blank
| Mail level | Leave blank
| Mail myhostname | Leave blank
| Mail from | Leave blank
| Mail to | Leave blank
| **Tuning**
| Max SSL Diffie-Hellman size | `2048`
| **Global Advanced pass thru**
| Custom options | Leave Blank
| **Configuration synchronization**
| HAProxy Sync | `☐  Sync HAProxy configuration to backup CARP members via XMLRPC.`

And click `Save`.

## 9.0 HAProxy Frontend Settings
All of the connection requests will be coming in to the same IP address and port but we need a way to distinguish between requests so that those for jellyfin-site1.foo.bar go to the jellyfin backend and those for sonarr-site1.foo.bar go to the sonarr backend.

So we will create a shared front end and then sub front ends for each subdomain.

### 9.1 Shared Frontend
In the pfSense WebGUI go to `Service` > `HAProxy` > `Frontend Tab` and click `Add` and fill out the necessary fields as follows:

| Edit HAProxy Frontend | Value 
| :--- | :--- 
| Name | `shared-frontend`
| Description | `Shared Frontend`
| Status `Active`
| **External address**
| Listen address | `WAN address (IPv4)`
| Custom address | Leave blank
| Port `443`
| SSL offloading | `☑` 
| Advanced | Leave blank
| Max Connections | `500`
| Type | `http / https(offloading)`
| **Default backend, access control lists and actions**
| Access Control lists | Leave blank
| Actions | Leave blank
| Default Backend | `None`
| **Stats options**
| Separate sockets | `☐ Enable collecting & providing separate statistics for each socket.`
| **Logging options**
| Don't log null | `☐ A connection on which no data has been transferred will not be logged.`
| Don't log normal | `☐ Don't log connections in which no anomalies are found.`
| Raise level for errors | `☐ Change the level from "info" to "err" for potentially interesting information.`
| Detailed logging | `☐ If checked provides more detailed logging.`
| **Error files**
| Error files | Leave blank
| **Advanced settings**
| Client timeout | Leave blank
| Use "forwardfor" option | `☑ Use "forwardfor" option.`
| Use "httpclose" option | `http=keep-alive (default)`
| Bind pass thru | Leave blank
| Advanced pass thru | Leave blank
| ** SSL Offloading**
| SNI filter | Leave blank
| Certificate | `site1.foo.bar (CA: Acmecert: 0=Let’s Encrypt,CN=Let’s Encrypt Authority X3,C=US)[Server cert]`
| Add ACL for certificate CommonName | `☑`
| Add ACL for certificate Subject Alternative Names | `☑`
| OCSP | `☐  Load certificate ocsp responses for easy certificate validation by the client.`
| Additional certificates | Leave blank
| Add ACL for certificate CommonName | `☐`
| Add ACL for certificate Subject Alternative Names | `☐`
| Advanced ssl options | Leave blank
| Advanced certificate specific ssl options | Leave blank
| **SSL Offloading - client certificates**
| Without client cert | `☐ Allows clients without a certificate to connect`
| Allow invalid cert | `☐ Allows client with a invalid/expired/revoked or otherwise wrong certificate to connect`
| Client verification CA certificates | Leave blank
| Client verification CRL | Leave blank

And click `Save`.

### 9.2 Jellyfin authentication Frontend
In the pfSense WebGUI go to `Service` > `HAProxy` > `Frontend Tab` and click `Add` and fill out the necessary fields as follows:

| Edit HAProxy Frontend | Value
| :--- | :--- 
| Name | `jellyfin-site1.foo.bar`
| Description | `Jellyfin authenticated frontend`
| Status `Active`
| Shared Frontend | `☑`
| Primary frontend | `shared-frontend - http`
| **Default backend, access control lists and actions**
| **Access Control lists**
| Table-Name | `jellyfin-acl`
| Table-Expresssion | `Host matches:`
| Table-CS | `☐`
| Table-Not | `☐`
| Table-Value| `jellyfin-site1.foo.bar`
| **Actions**
| Table-Action | `Use Backend`
| Table-Parameters | Leave blank
| Table-Conditions acl names | `jellyfin-acl`
| Default Backend | `None`
| **Error files**
| Error files | Leave blank
| **SSL Offloading**
| Use offloading | `☐ Specify additional certificates for this shared-frontend`

And click `Save`.

### 9.3 Sonarr authentication Frontend
In the pfSense WebGUI go to `Service` > `HAProxy` > `Frontend Tab` and click `Add` and fill out the necessary fields as follows:

| Edit HAProxy Frontend | Value
| :--- | :---
| Name | `sonarr-site1.foo.bar`
| Description | `Sonarr authenticated frontend`
| Status `Active`
| Shared Frontend | `☑`
| Primary frontend | `shared-frontend - http`
| **Default backend, access control lists and actions**
| **Access Control lists**
| Table-Name | `sonarr-acl`
| Table-Expresssion | `Host matches:`
| Table-CS | `☐`
| Table-Not | `☐`
| Table-Value| `sonarr-site1.foo.bar`
| **Actions**
| Table-Action | `Use Backend`
| Table-Parameters | Leave blank
| Table-Conditions acl names | `sonarr-acl`
| Default Backend | `None`
| **Error files**
| Error files | Leave blank
| **SSL Offloading**
| Use offloading | `☐ Specify additional certificates for this shared-frontend`

And click `Save`.

### 9.4 Radarr authentication Frontend
In the pfSense WebGUI go to `Service` > `HAProxy` > `Frontend Tab` and click `Add` and fill out the necessary fields as follows:

| Edit HAProxy Frontend | Value
| :--- | :---
| Name | `radarr-site1.foo.bar`
| Description | `Radarr authenticated frontend`
| Status `Active`
| Shared Frontend | `☑`
| Primary frontend | `shared-frontend - http`
| **Default backend, access control lists and actions**
| **Access Control lists**
| Table-Name | `radarr-acl`
| Table-Expresssion | `Host matches:`
| Table-CS | `☐`
| Table-Not | `☐`
| Table-Value| `radarr-site1.foo.bar`
| **Actions**
| Table-Action | `Use Backend`
| Table-Parameters | Leave blank
| Table-Conditions acl names | `radarr-acl`
| Default Backend | `None`
| **Error files**
| Error files | Leave blank
| **SSL Offloading**
| Use offloading | `☐ Specify additional certificates for this shared-frontend`

And click `Save`.

### 9.5 Nzbget authentication Frontend
In the pfSense WebGUI go to `Service` > `HAProxy` > `Frontend Tab` and click `Add` and fill out the necessary fields as follows:

| Edit HAProxy Frontend | Value
| :--- | :---
| Name | `nzbget-site1.foo.bar`
| Description | `Nzbget authenticated frontend`
| Status `Active`
| Shared Frontend | `☑`
| Primary frontend | `shared-frontend - http`
| **Default backend, access control lists and actions**
| **Access Control lists**
| Table-Name | `nzbget-acl`
| Table-Expresssion | `Host matches:`
| Table-CS | `☐`
| Table-Not | `☐`
| Table-Value| `nzbget-site1.foo.bar`
| **Actions**
| Table-Action | `Use Backend`
| Table-Parameters | Leave blank
| Table-Conditions acl names | `nzbget-acl`
| Default Backend | `None`
| **Error files**
| Error files | Leave blank
| **SSL Offloading**
| Use offloading | `☐ Specify additional certificates for this shared-frontend`

And click `Save`.

### 9.6 Deluge authentication Frontend
In the pfSense WebGUI go to `Service` > `HAProxy` > `Frontend Tab` and click `Add` and fill out the necessary fields as follows:

| Edit HAProxy Frontend | Value
| :--- | :---
| Name | `deluge-site1.foo.bar`
| Description | `Deluge authenticated frontend`
| Status `Active`
| Shared Frontend | `☑`
| Primary frontend | `shared-frontend - http`
| **Default backend, access control lists and actions**
| **Access Control lists**
| Table-Name | `deluge-acl`
| Table-Expresssion | `Host matches:`
| Table-CS | `☐`
| Table-Not | `☐`
| Table-Value| `deluge-site1.foo.bar`
| **Actions**
| Table-Action | `Use Backend`
| Table-Parameters | Leave blank
| Table-Conditions acl names | `deluge-acl`
| Default Backend | `None`
| **Error files**
| Error files | Leave blank
| **SSL Offloading**
| Use offloading | `☐ Specify additional certificates for this shared-frontend`

And click `Save`.

### 9.7 Ombi authentication Frontend
In the pfSense WebGUI go to `Service` > `HAProxy` > `Frontend Tab` and click `Add` and fill out the necessary fields as follows:

| Edit HAProxy Frontend | Value
| :--- | :---
| Name | `ombi-site1.foo.bar`
| Description | `Ombi authenticated frontend`
| Status `Active`
| Shared Frontend | `☑`
| Primary frontend | `shared-frontend - http`
| **Default backend, access control lists and actions**
| **Access Control lists**
| Table-Name | `ombi-acl`
| Table-Expresssion | `Host matches:`
| Table-CS | `☐`
| Table-Not | `☐`
| Table-Value| `ombi-site1.foo.bar`
| **Actions**
| Table-Action | `Use Backend`
| Table-Parameters | Leave blank
| Table-Conditions acl names | `ombi-acl`
| Default Backend | `None`
| **Error files**
| Error files | Leave blank
| **SSL Offloading**
| Use offloading | `☐ Specify additional certificates for this shared-frontend`

And click `Save`.

### 9.8 Syncthing authentication Frontend
In the pfSense WebGUI go to `Service` > `HAProxy` > `Frontend Tab` and click `Add` and fill out the necessary fields as follows:

| Edit HAProxy Frontend | Value
| :--- | :---
| Name | `syncthing-site1.foo.bar`
| Description | `Syncthing authenticated frontend`
| Status `Active`
| Shared Frontend | `☑`
| Primary frontend | `shared-frontend - http`
| **Default backend, access control lists and actions**
| **Access Control lists**
| Table-Name | `syncthing-acl`
| Table-Expresssion | `Host matches:`
| Table-CS | `☐`
| Table-Not | `☐`
| Table-Value| `syncthing-site1.foo.bar`
| **Actions**
| Table-Action | `Use Backend`
| Table-Parameters | Leave blank
| Table-Conditions acl names | `syncthing-acl`
| Default Backend | `None`
| **Error files**
| Error files | Leave blank
| **SSL Offloading**
| Use offloading | `☐ Specify additional certificates for this shared-frontend`

And click `Save`.

## 10.0 HAProxy Backend Settings
HAProxy backend section defines a group of servers that will be assigned to handle requests. A backend server responds to incoming requests if a given condition is true.

### 10.1 Jellyfin Backend
In the pfSense WebGUI go to `Service` > `HAProxy` > `Backend Tab` and click `Add` and fill out the necessary fields as follows:

| Edit HAProxy Backend server pool | Value
| :--- | :---
| Name | `jellyfin-site1.foo.bar`
| Server list
| Table-Mode | `active`
| Table-Name | `jellyfin`
| Table-Forwardto | `Address+Port`
| Table-Address | `192.168.50.111`
| Table-Port | `8096`
| Table-Encrypt(SSL) | `☐`
| Table-SSL checks | `☐`
| Table-Weight | Leave blank
| **Loadbalancing options (when multiple servers are defined)**
| Balance | `☑ None`
| **Access control lists and actions**
| Access Control lists | Leave blank
| Actions | Leave blank
| **Timeout / retry settings**
| Connection timeout | Leave blank
| Server timeout | Leave blank
| Retries | Leave blank
| **Health checking**
| Health check method | `Basic`
| Check frequency | Leave blank
| Log checks | `☑ When this option is enabled, any change of the health check status or to the server's health will be logged.`
| **Agent checks**
| Agent checks | `☐ Use agent checks`
| **Cookie persistence**
| Cookie Enabled | `☐ Enables cookie based persistence. (only used on "http" frontends)`
| **Stick-table persistence**
| Stick tables | `none`
| **Email notifications**
| Mail level | `Default level from global`
| Mail to | Leave blank
| **Statistics**
| Stats Enabled | `☐ Enables the haproxy statistics page (only used on "http" frontends)`
| **Error files**
| Error files | Leave blank
| **HSTS / Cookie protection**
| HSTS Strict-Transport-Security | Leave blank
| Cookie protection | `☐ Set "secure" attribure on cookies (only used on "http" frontends)`
| **Advanced settings**
| Per server pass thru | Leave blank
| Backend pass thru | Leave blank
| Transparent ClientIP | `☐  Use Client-IP to connect to backend servers`

And click `Save`.

Repeat for all your backend servers. To make life easy you can click the `Copy` icon under `Actions` to duplicate your first backend entry (i.e jellyfin-site1.foo.bar). Then edit the copied entry/duplicate changing the following particulars to meet other server needs:

| Edit HAProxy Backend server pool | Jellyfin Value | Sonarr Value | Radarr Value | Nzbget Value | Deluge Value | Ombi Value | Syncthing Value
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :---
| Name | `jellyfin-site1.foo.bar` | `sonarr-site1.foo.bar` | `radarr-site1.foo.bar` | `nzbget-site1.foo.bar` | `deluge-site1.foo.bar` | `ombi-site1.foo.bar` | `syncthing-site1.foo.bar`
| Server list
| Table-Mode | `active` | `active` | `active` | `active` | `active` | `active` | `active` 
| Table-Name | `jellyfin` | `sonarr` | `radarr` | `nzbget` | `deluge` | `ombi` | `syncthing`
| Table-Forwardto | `Address+Port` | `Address+Port` | `Address+Port` | `Address+Port` | `Address+Port` | `Address+Port` | `Address+Port`
| Table-Address | `192.168.50.111` | `192.168.50.112` | `192.168.50.113` | `192.168.50.114` | `192.168.50.115` | `192.168.50.119` | `192.168.50.122`
| Table-Port | `8096` | `8989` | `7878` | `8080` | `8112`| `5000` | `8384`
| Remaining fields are common

And click `Save`.

## 11.0 Fix for pfSense Dynamic DNS
If your ISP frequently changes your WAN IP you may run into problems with out of date Cloudfare A-records pointing to an out of date IP address.

It appears updates may take place if the WAN interface IP address changes, but not if the pfSense device is behind a router, gateway or firewall.

You will know if you have problem when you cannot remotely access your server node, the pfSense `Services` > `Dynamic DNS` > `Dynamic DNS Clients` page shows cached IP addresses in red indicating that pfSense knows the cashed IP address is not the current public WAN IP and that is has not updated the Dynamic DNS host (Cloudfare) with the current public WAN IP. 

The work around is to install a CRON manager.

### 11.1 Install a Cron Manager
In the pfSense WebGUI go to `System` > `Package Manager` > `Available Packages Tab` and search for `Cron`. Install the `Cron` package.

### 11.2 Configure your Dynamic DNS Cron Schedule
In the pfSense WebGUI go to `Services` > `Cron` > `Settings Tab` and click on the pencil for entry with `rc.dyndns.update` in its command name.  Edit the necessary fields as follows:

| Add A Cron Schedule | Value
| :--- | :---
| Minute | `*/5`
| Hour | `*`
| Day of the Month | `*`
| Month of the Year | `*`
| Day of the Week | `*`
| User | `root`
| Command | Leave default

And click `Save`.

This will force pfsense tocheck for WAN IP changes every 5 minutes.

## 00.00 Patches & Fixes
Tweaks and fixes to make broken things work - sometimes.

### 00.10 pfSense Dynanic DNS Cloudflare with proxy enabled doesn't work at all
Fix for log errors like this:

```
 PAYLOAD: {"success":false,"errors":[{"code":1004,"message":"DNS Validation Error","error_chain":[{"code":9003,"message":"Invalid 'proxied' value, must be a boolean"}]}],"messages":[],"result":null}
 ```
 
In the pfSense WebGUI go to `Diagnostics` > `Edit File` > `Browse Tab` and browse to folder /etc/inc/ and select `services.inc` file. Now use the `GoTo line` field and type in `1881`. Replace `$dnsProxied = $conf['proxied'],` with `$dnsProxied = isset($conf['proxied']),` and click `Save`. Reboot pfSense.

```
	$dns = new updatedns($dnsService = $conf['type'],
		$dnsHost = $conf['host'],
		$dnsDomain = $conf['domainname'],
		$dnsUser = $conf['username'],
		$dnsPass = $conf['password'],
		$dnsWildcard = $conf['wildcard'],
                $dnsProxied = isset($conf['proxied']),
		$dnsMX = $conf['mx'],
```

