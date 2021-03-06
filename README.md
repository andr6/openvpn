# OpenVPN with ScrambleSuit and DNS
This [ansible](http://docs.ansible.com/ansible/latest/intro_installation.html) script will allow you to install from scratch your own [OpenVPN](https://openvpn.net/index.php/open-source.html) server with [scramblesuit](https://www.cs.kau.se/philwint/scramblesuit/) and private [DNS](https://www.isc.org/downloads/bind/) server within minutes. Level of knowledge required: **basic**

There is no bul**hit, no unessesery clunky software, it's based on [OpenBSD](http://www.openbsd.org), simple ansible playbook, easy as any kid can read. 
Once playbook finish, you have ready to use 2 archives with configs and all what is needed to connect to your VPN: one config is for Desktop Viscosity app and second for iPhone OpenVPN app (_ovpn_). You can easly create more keypairs/config for more users and adapt to your needs. Really simple, see below for usage.

## Why ?
Becouse other solutions are crap. So called "_private_" VPNs that are sold are no private - you let **unknown party** to watch _all_ your traffic, they sell it to Ad companies or do what they want with _your_ data. It's really stupid and people are unaware of this.
This playbook guaranetee that your data on transit are safe, server _do not_ store anything related with traffic or DNS queries, even in unlikely breach to your VPN server attacker wont be able to do anything that could harm your data (_of course once you realize server was pwned_). Read below why using VPN on your mobile and desktop is important.

## Security

I am using Easy-RSA 3 to setup [PKI](https://en.wikipedia.org/wiki/Public_key_infrastructure), it's easy to manage (*see below*). [ECC](https://en.wikipedia.org/wiki/Elliptic_curve_cryptography) keypairs use *secp521r1* curve (*which is perhaps overkill, and you can safely lower it to more efficient secp256k1*), and RSA uses 2048 bit keys with SHA256 signatures. 

Mobile connections uses **DHE-RSA-AES256-SHA** TLS1.2 for control channel and **AES-256-CBC** for data encryption, also **HMAC** is used for packets authentication.

Desktop connections use **ECDHE-ECDSA-AES256-GCM-SHA384** TLS1.2 for control channel and **AES-256-GCM** for data encryption, in additions openvpn is configured to use **tls-crypt** with symetric key for packet encryption and authentication.

Additionally Desktop connections are wrapped with scrabmlesuit tunnel (*part of Tor project*) and although some say that's not needed when useing *tls-crypt* some think otherwise ;) ... anyway, this is how it is done here.

There are other settings that ensure connection is safe, like EKU, CA hash verification and others, see config for details. 

Last thing on the list is **DNS** server that is setup with this playbook. It's **Bind9** with [DNSSEC](https://en.wikipedia.org/wiki/Domain_Name_System_Security_Extensions) resolver enabled. This ensures that your queries do not leak to other providers and you always use legacy (*your own*) DNS server. I *do not use* Google DNS or other crap caching servers like OpenDNS (*who btw strip DNS records from DNSSEC signatures - which simply speking can be seen as fraudulent itself...*). You can verify DNS leaks on site like: [https://www.dnsleaktest.com](https://www.dnsleaktest.com) and on [https://dnssec.vs.uni-due.de](https://dnssec.vs.uni-due.de) verify if DNSSEC resolver works as expected.

## My choose of cloud provider, apps and why

For this playbook i have choosen [exoscale](https://www.exoscale.ch) as cloud provider (*but it will run on any OpenBSD you choose*). 
Why exoscale? Becouse it's Swiss, it's independent from US influences and obey only Swiss law, also they are nice and simply to use. Also their prices are quite low - or comparable to other's like DigitalOcean or AWS. Performance of the single CPU core is sufficent for OpenVPN in [Micro](https://portal.exoscale.ch/register?r=gLrEOdv5hVgv) instance do not use anything bigger then that as long as you do not use it for over 10 users.

If you are going to use **exoscale** please use my **invite code** ( gLrEOdv5hVgv ), or [this link](https://portal.exoscale.ch/register?r=gLrEOdv5hVgv) - you will get **50 CHF** credit after *second* payment - that's ammount that will let you use VPN server for free for next 5 months !!


For the desktop client side, i recommend using [Viscosity VPN](https://www.sparklabs.com/viscosity/) - no freebies here ;) - is it easy to use OpenVPN client that works on Windows and MacOS. It is well developed and uses recent openvpn client software.

If you are using iPhone, config is generated for free app [OpenVPN Connect](https://itunes.apple.com/pl/app/openvpn-connect/id590379981?l=pl&mt=8) - only one legacy app. I assume there are some apps for Android as well but i do not have one so cannot recommend any...

## How to use playbook

Below simple requirements to run your own VPN server

#### Requirements
* have ansible installed on your computer
* have running newly created OpenBSD instance in some cloud provider (*here we use [exoscale](https://portal.exoscale.ch/register?r=gLrEOdv5hVgv) as stated above*)
* allow SSH port 22 for install from your host, and permamently allow TCP 80 and 443 for VPN
* basic knowledge of using terminal and ssh
* pretty much that's all

#### Steps to start your own OpenVPN server from ansible playbook:
* edit *private_vpn_inventory* and replace **IP_OF_YOUR_SERVER** with IP of your cloud server (easy?)
* run ansible with command: **ansible-playbook -i private_vpn_inventory openvpn.yml**
* after ansible finish without error your server is ready to use

get your configs, they are in
* **/etc/openvpn/export/archives/privateVPN-Desktop-JohnDoe.tar.gz** - this is for Viscosity DesktopApp
* **/etc/openvpn/export/archives/privateVPN-Mobile-JohnDoe.tar.gz** - this is for iPhone OpenVPN app

you can use: *scp root@SERVER_IP:/etc/openvpn/export/archives/\* .* to copy config files all at once.

Once all is done, you can import above configs into your Viscosity app or/and iPhone OpenVPN app - no changes required all is already set. 

#### Generating additional certificates for users

If more users are going to use OpenVPN then you need to generate new key-pairs (*each for each user*). This is simple to do and there are 2 ways of doing it:

* create ansible play - this is more advanced and i will not cover it here
* use **gen_config.sh**, steps below:

on the server, go to: */etc/openvpn/easy-rsa/* and type:

**./easyrsa --use-algo=rsa build-client-full privateVPN-Mobile-USERNAME nopass** - for *Mobile* client, note that part "privateVPN-Mobile-" shoule be unchanged in certificate name, just add proper USERNAME (*no spaces or crazy stuff here, just a-azA-Z*). *Mobile* is a keyword used later by script.

**./easyrsa --use-algo=ec --curve=secp521r1 build-client-full privateVPN-Desktop-USERNAME nopass** - for *Desktop* client, note that part "privateVPN-Desktop-" should be unchanged in certificate name, just add proper USERNAME, same as above - no crazy characters. *Desktop* is a keyword, do not change it.

**Note:** as you can see private keys are generated *without* password, you can password-protect them by removing **nopass** option. You will be asked for password and this is **recommended** way of generating keypair. I use nopass just for the convenience of the playbook. Also, for god sake **do not send keypairs via email or any other crazy way** without properly encrypting them, best - set password on key and wrap up by some gpg.

Once you understood all, let's generate packages with config, easy like 1,2,3...: go to: */etc/openvpn/export/* and symlink all new keypairs into export folder: **ln -s /etc/openvpn/easy-rsa/pki/issued/privateVPN\* .** and **ln -s /etc/openvpn/easy-rsa/pki/private/privateVPN\* .** then for each user run: **./gen_config.sh privateVPN-Desktop-USERNAME** packages are put into *archives/* folder. Copy to localhosts, share, install, enjoy.

That's all. 


#### Client Configuration

Desktop config creates 172.17.200.0/24 network, access on port 80

Mobile config creates 172.16.200.0/24 network, access on port 443


#### Customizations

ToDo

## Legal Warning

Remember: This do not give you privacy in internet, this playbook was made primarly to make connection to the internet more safe - especially on mobile devices. Certainty using VPN do not give you right to break law - when you do - no VPN can save you - you will be found and prosecuted according to the law. Don't be stupid, think and do wise things. Be repectfull for other people, do not be a jerk.

#### This doc is work-in-progress
