# HAProxy in pfSense as a Reverse Proxy
If you want convenient remote access to your LXC's and/or Apps the easiest way is to setup HAProxy application addon in pfSense. Then you will have access to your Apps using address URLS like:
>  unifi-site1.foo.bar --> unifi 192.168.1.251
>  appname-site1.foo.bar --> appname 192.168.1.XXX

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
| `A` | `sabnzbd-site1` | 0.0.0.0 | `Automatic TTL` | `OFF` |
| `A` | `deluge-site1` | 0.0.0.0 | `Automatic TTL` | `OFF` |
| `A` | `vpn-site1` | 0.0.0.0 | `Automatic TTL` | `OFF` |

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
| **jellyfin.site1.foo.bar**
| Disable | `☐` Disable this client |*Uncheck*
| Service Type | `Cloudfare`
| Interface to monitor | `WAN`
| Hostname | `jellyfin.site1`
| Domain | `foo.bar` | *Replace with your domain name.*
| MX | Leave Blank
| Wildcards | `☑` Enable Wildcard
| Cloudflare Proxy | Enable Proxy
| Verbose logging | `☑` Enable verbose logging
| Username | Enter your Cloudfare Accounts reg'd Email Address
| Password | Enter your Global API Key | *See section 3.0*
| Description | `jellyfin-site1.foo.bar`


## 4.0 Install ACME on pfSense
We need to install the ACME package on your pfSense. ACME is Automated Certificate Management Environment, for automated use of LetsEncrypt certificates.

In the pfSense WebGUI go to `System` > `Package Manager` > `Available Packages Tab` and search for `ACME`. Install the `ACME` package.

## 5.0 Generate ACME Certificates
We will need to generate certificates from a trusted provider such as Let’s Encrypt and a few from within pfSense itself.

We need 2x wildcard certificates from Let’s Encrypt. Creating wildcard certificates will allow us to create subdomains without having to generate a new certificate for each one.

We can use the ACME Package provided in pfSense.

### 5.1 Create ACME Account Keys
First you need to create some account keys. LetsEncrypt is rate limited so you want to make sure that you have everything configured correctly before requesting a real cert. To help people test, LetsEncrypt provides a test service that you can use as you figure out your settings without bumping into the rate limit on the production servers. Certs obtained from these test services cannot be used.

So we will create two Account Keys.

In the pfSense WebGUI go to `Services` > `Acme Certificates` > `Account Keys`. Click `Add` and fill out the necessary fields as follows:

| Edit Certificate options | Value | Notes
| :--- | :---
| **Production Key**
| Name | `site1.foo-production` | *For example, `site1.foo-production`*
| Description | `site1.foo-production key` | * For example, `site1.foo-production key`*
| Acme Server | `Let’s Encrypt Production ACME v2 (Applies rate limits to certificate requests)`
| E-Mail Address | Enter your email address
| Account key | `Create new account key` | *Click `Create new account key`*
| Acme account registration | `Register acme account key` | *Click `Register acme account key`*
| **Test Key**
| Name | `site1.foo-test` | *For example, `site1.foo-test`*
| Description | `site1.foo-test key` | * For example, `site1.foo-test key`*
| Acme Server | `Let’s Encrypt Staging ACME v2 (for TESTING purposes)`
| E-Mail Address | Enter your email address
| Account key | `Create new account key` | *Click `Create new account key`*
| Acme account registration | `Register acme account key` | *Click `Register acme account key`*

It is really important that you choose the Staging ACME v2 server. Only the v2 will support wildcard domains.

Now click the `+ Create new account key` button and wait for the box to fill in with a new RSA private key.

Then click the `Register ACME Account key`. The little cog will spin and if it worked the cog will turn into a check. 

Finally click `Save`.

### 5.2 Create ACME Certificates
In the pfSense WebGUI go to `Services` > `Acme Certificates` > `Certificates`. Click `Add` and fill out the necessary fields as follows. Notice have two entries in the Domain SAN List. Every level of the domain needs to have it’s own certificate. So we will get a certificate that covers both foo.bar and *.foo.bar.

| Edit Certificate options | Value | Notes
| :--- | :--- |:---
| Name | `wildcard.site1.foo.bar`
| Description | `Wildcard for site1.foo.bar`
| Status | `active`
| Acme Account | `site1.foo-test`
| Private Key | `2048-bit RSA`
| OCSP Must Staple | `☐ Add the OCSP Must Staple extension to the certificate`
| **Entry 1 - Domain SAN list**
| Mode |`Enabled`
| Domainname `site1.foo.bar` | *No wildcard for this entry*
| Method | `DNS-Cloudflare`
| Mode | `Enabled`
| Key | Fill in the Cloudfare Global API Key
| Email | Enter your email addresss
| Enable DNS alias mode | Leave Blank
| Enable DNS domain alias mode | `☐ (Optional) Uses the challenge domain alias value as --domain-alias instead in the acme.sh call`
| **Entry 2 - Domain SAN list**
| Mode |`Enabled`
| Domainname `*.site1.foo.bar` | *This entry is the wildcard of site1.foo.bar*
| Method | `DNS-Cloudflare`
| Mode | `Enabled`
| Key | Fill in the Cloudfare Global API Key
| Email | Enter your email addresss
| Enable DNS alias mode | Leave Blank
| Enable DNS domain alias mode | `☐ (Optional) Uses the challenge domain alias value as --domain-alias instead in the acme.sh call`
| DNS Sleep | Leave Blank
| Action List
| Last renewal | Leave Blank
| Certificate renewal after | `90`

Then click `Save`  followed by `Issue/Renew`. A review of the output will appear on the page and if successful you see a RSA key has been generated. The output should begin like this:

```
wildcard.site1.foo.bar
Renewing certificate
account: foo-test
server: letsencrypt-staging-2


/usr/local/pkg/acme/acme.sh --issue -d 'site1.foo.bar' --dns 'dns_cf' -d '*.site1.foo.bar' --dns 'dns_cf' --home '/tmp/acme/wildcard.site1.foo.bar/' --accountconf '/tmp/acme/wildcard.site1.foo.bar/accountconf.conf' --force --reloadCmd '/tmp/acme/wildcard.site1.foo.bar/reloadcmd.sh' --log-level 3 --log '/tmp/acme/wildcard.site1.foo.bar/acme_issuecert.log'

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
[Sat Aug 3 14:34:58 +07 2019] Multi domain='DNS:site1.foo.bar,DNS:*.site1.foo.bar'
[Sat Aug 3 14:34:58 +07 2019] Getting domain auth token for each domain
[Sat Aug 3 14:35:12 +07 2019] Getting webroot for domain='site1.foo.bar'
[Sat Aug 3 14:35:12 +07 2019] Getting webroot for domain='*.site1.foo.bar'
[Sat Aug 3 14:35:12 +07 2019] site1.foo.bar is already verified, skip dns-01.
[Sat Aug 3 14:35:12 +07 2019] *.site1.foo.bar is already verified, skip dns-01.
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

## 6.0 Create Certificate Authorities
In the pfSense WebGUI go to `System` > `Certificate Manager` > `CAs`. Click `Add` and fill out the necessary fields as follows:

| Create / Edit CA | Value
| :--- | :--- 
| Descriptive name | `site1.myserver Remote Access`
| Method | `Create an internal Certificate Authority`
| **Internal Certificate Authority**
| Key Length (bits) | `4096`
| Digest Algorithm | `SHA512`
| Lifetime (days) | `3650`
| Common Name | internal-ca
| **The following certificate authority subject components are optional and may be left blank**
| Country code | Choose your country
| State | Type your State
| City | Type your City
| Organization | Leave Blank
| Email address | Enter your email address
| Common Name | `site1.myserver.com VPN Remote Access`

And click `Save`.

## 7.0 Internal Certificates
In the pfSense WebGUI go to `System` > `Certificate Manager` > `Certificates`. Click `Add/Sign` and fill out the necessary fields as follows:

| Add/Sign a New Certificate | Value
| :--- | :---
| Method | `Create an internal Certificate`
| Descriptive name | `vpn.site1.myserver.com`
| **Internal Certificate**
| Certificate authority | site1.myserver.com VPN Remote Access
| Key Length | `4096`
| Digest Algorithm | `SHA512`
| Lifetime (days) | `3650`
| Common Name | vpn.site1.myserver.com
| Country code | Leave as Default
| State | Leave as Default
| City | Leave as Default
| Organization | Leave as Default
| Organisational Unit | site1.myserver.me VPN Remote Access
| **Certificate Attributes**
| Attribute Notes
| Certificate Type | `Server Certificate`
| Alternate Names | Leave as Default

And click `Save`.

## 8.0 User Certificates
You can create user certificates for each user that will be allowed to access to your VPN features.

In the pfSense WebGUI go to `System` > `Certificate Manager` > `Certificates`. Click `Add/Sign` and fill out the necessary fields as follows:

| Add/Sign a New Certificate | Value
| :--- | :---
| Method | `Create an internal Certificate`
| Descriptive name | `vpn.site1.myserver.com Username`
| **Internal Certificate**
| Certificate authority | site1.myserver.com VPN Remote Access
| Key Length | `4096`
| Digest Algorithm | `SHA512`
| Lifetime (days) | `3650`
| Common Name | `vpn.site1.myserver.com Username`
| Country code | Leave as Default
| State | Leave as Default
| City | Leave as Default
| Organization | Leave as Default
| Organisational Unit | site1.myserver.me VPN Remote Access
| **Certificate Attributes**
| Attribute Notes
| Certificate Type | `Server Certificate`
| **Alternate Names**
| Type | `email address`
| Value | Enter Usernames email address

And click `Save`.

## 9.0 Install HAProxy






