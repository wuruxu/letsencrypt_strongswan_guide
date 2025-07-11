# letsencrypt_strongswan_guide

[![letsencrypt](res/images/letsencrypt-logo-horizontal.png)](https://letsencrypt.org/) ![puls](res/images/add-symbolic.symbolic.png) [![strongswan](res/images/strongswan.png)](https://strongswan.org/)           
##Requirement
* A domain name: for example **xyz.wuruxu.cn** and resolve to VPS public IP
* VPS: [digitalocean droplets](https://m.do.co/c/e7e1a9f66991)
* Ports 4500/UDP, 500/UDP, 51/UDP and 50/UDP opened in the firewall

## 1. Setup VPN Server
### 1.1 Build strongswan
 openssl >= 1.0.2 is required for enable ECP521/ECP384/ECP256 & ECDSA support
```
# apt-get install libssl-dev
```
```
#./configure --prefix=/usr --sysconfdir=/etc --localstatedir=/var --enable-af-alg --enable-ccm --enable-chapoly --enable-ctr --enable-gcm --enable-newhope --enable-openssl --enable-aesni --enable-sqlite --enable-dhcp --enable-eap-identity --enable-eap-tls --enable-eap-ttls --enable-eap-mschapv2 --enable-systemd CFLAGS=-O2 --disable-ikev1
```
### 1.2 Issue free certificate (RSA or ECDSA)

#### 1.2.1 clone [acme.sh](https://github.com/Neilpang/acme.sh) to issue certificate
```
# git clone https://github.com/Neilpang/acme.sh
```

##### [ECDSA certiifcate](https://en.wikipedia.org/wiki/Elliptic_Curve_Digital_Signature_Algorithm) - recommend
```
# ./acme.sh --issue --standalone -d xyz.wuruxu.cn --keylength ec-384 --server letsencrypt

```
##### [RSA certificate](https://en.wikipedia.org/wiki/RSA_(cryptosystem))
```
# ./acme.sh --issue --standalone -d xyz.wuruxu.cn --keylength 4096 --server letsencrypt
```
#### 1.2.2 install certificates in strongswan
```
# cp ca.cer /etc/ipsec.d/cacerts/
# cp fullchain.cer  /etc/ipsec.d/certs/acme_xyz_server.cert.pem
# cp xyz.wuruxu.cn.key  /etc/ipsec.d/private/acme_xyz_ecc.pem
```
###1.3 Configure strongswan server
in ipsec.conf, left as 'local', right as 'remote'
##### 1.3.1 /etc/ipsec.conf
```
# ipsec.conf - strongSwan IPsec configuration file

config setup
  uniqueids=never

conn %default
  keyexchange=ikev2
  left=%defaultroute
  leftauth=pubkey
  leftfirewall=yes
  right=%any
  mobike=yes
  compress=no
  ike=chacha20poly1305-sha512-newhope128,chacha20poly1305-sha512-x25519,aes256-sha512-modp2048,aes128-sha512-modp2048,aes256ccm96-sha384-modp2048,aes256-sha256-modp2048,aes128-sha256-modp2048,aes128-sha1-modp2048!
  esp=chacha20poly1305,aes256gcm128,aes128gcm128,aes256ccm128,aes256
  
  
 # IPv4 connection server config
 conn ec4
  leftsendcert=always
  leftcert=acme_xyz_server.cert.pem
  leftid=xyz.wuruxu.cn //same as certificate domain name
  leftsubnet=0.0.0.0/0
  rightauth=eap-mschapv2
  rightsourceip=10.18.0.0/24
  rightsendcert=never
  eap_identity=%any
  auto=add
 
# IPv6 connection server config
conn ec6
  leftsendcert=always
  leftcert=nginx.ssl.ipv6.wuruxu.ecc.cer
  leftid=@ipv6.wuruxu.cn
  leftsubnet=0.0.0.0/0,::/0
  rightauth=eap-mschapv2
  rightsourceip=2001:0909:1818:2d18:1::/80,10.128.0.0/24
  rightdns=2001:4860:4860::8888,1.1.1.1
  rightsendcert=never
  eap_identity=%identity
  auto=add
```
#### 1.3.2 /etc/ipsec.secrets
```
: ECDSA acme_xyz_ecc.pem
user : EAP "userpasswd"
```
#### 1.3.3 /etc/sysctl.conf
```
net.ipv4.ip_forward = 1  
net.ipv4.conf.all.accept_redirects = 0  
net.ipv4.conf.all.send_redirects = 0  
```
enable sysctl rules
```
#  sysctl -p  
```
###1.4 Start strongswan
#### 1.4.1 apply iptables rule
```
# iptables -A INPUT -p udp --dport 500 --j ACCEPT
# iptables -A INPUT -p udp --dport 4500 --j ACCEPT
# iptables -A INPUT -p esp -j ACCEPT
# iptables -t nat -A POSTROUTING -s 10.18.0.0/24 -o eth0 -j MASQUERADE
```
#### 1.4.2 start ipsec daemon
```
# ipsec start
```
## 2. Strongswan Client Configuration
![support platfom](res/images/platform-all.png)      
* [Window 7+](https://wiki.strongswan.org/projects/strongswan/wiki/Windows7#C)
* [iOS & MAC OSX](https://wiki.strongswan.org/projects/strongswan/wiki/AppleClients)
* [Android 4+](https://wiki.strongswan.org/projects/strongswan/wiki/AndroidVPNClient)    
##### additional NOTES for android app to handle multi SSL certificate on one server:  #####
![Advance Config](res/images/android-strongswan-config.jpg)
* Linux       

#### 1. install strongswan on linux platform(same options as above)
```
./configure --prefix=/usr --sysconfdir=/etc --localstatedir=/var --enable-af-alg --enable-ccm --enable-chapoly --enable-ctr --enable-gcm --enable-newhope --enable-openssl --enable-aesni --enable-sqlite --enable-dhcp --enable-eap-identity --enable-eap-tls --enable-eap-ttls --enable-eap-mschapv2 --enable-systemd CFLAGS=-O2 --disable-ikev1
```
#### install DST Root CA
```
$ cp /etc/ssl/certs/DST_Root_CA_X3.pem /etc/ipsec.d/cacerts/
```
#### 2. config strongswan
##### 2.1 /etc/ipsec.secrets
```
# /etc/ipsec.secrets - strongSwan IPsec secrets file

user : EAP "userpasswd"

```
##### 2.2 /etc/ipsec.conf
```
# ipsec.conf - strongSwan IPsec configuration file

# basic configuration

config setup
  strictcrlpolicy=no
  uniqueids = never

conn %default
  ikelifetime=3h
  keylife=60m
  rekeymargin=9m
  keyingtries=3
  keyexchange=ikev2
  ike=chacha20poly1305-sha512-x25519,aes256-sha512-modp4096,aes128-sha512-modp4096,aes256ccm96-sha384-modp2048,aes256-sha256-modp2048,aes128-sha256-modp2048,aes128-sha1-modp2048!
  esp=aes256gcm128,aes128gcm128,aes256ccm128,aes256

conn ec4
  left=%any
  leftfirewall=yes
  leftauth=eap-mschapv2
  leftid=debian # specifiy the client ID
  leftsourceip=%config4
  leftikeport = 4500
  eap_identity=wuruxu  # user in /etc/ipsec.secrets
  right=xyz.wuruxu.cn
  rightauth=pubkey
  rightid=xyz.wuruxu.cn
  rightsubnet=0.0.0.0/0
  auto=add
  
conn net0
  leftsubnet=192.168.0.0/16
  rightsubnet=192.168.0.0/16
  authby=never
  type=pass
  auto=route

conn net1
  leftsubnet=10.0.0.0/8
  rightsubnet=10.0.0.0/8
  authby=never
  type=pass
  auto=route
```
##### 2.3 start up connection ec3
```
# ipsec start
# ipsec up ec4
```
##### NOTES: mtu size issue on linux platform #####
edit /etc/strongswan.d/charon/kernel-netlink.conf to change mtu size from 0 to 1280
```
mtu = 1280
```
