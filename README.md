# letsencrypt_strongswan_guide

[![letsencrypt](res/images/letsencrypt-logo-horizontal.png)](https://letsencrypt.org/) ![puls](res/images/add-symbolic.symbolic.png) [![strongswan](res/images/strongswan.png)](https://strongswan.org/)           
##Requirement
* A domain name: for example **xyz.wuruxu.com** and resolve to VPS public IP
* VPS: [Linode VPS](https://www.linode.com/?r=0bc6a0c838d110075a691b29f2c49d9e90ce2eed)
* Ports 4500/UDP, 500/UDP, 51/UDP and 50/UDP opened in the firewall

## 1. Setup VPN Server
### 1.1 Build strongswan
 openssl >= 1.0.2 is required for enable ECP521/ECP384/ECP256 & ECDSA support
```
# apt-get install libssl-dev
```
```
#./configure --prefix=/usr --sysconfdir=/etc --localstatedir=/var CFLAGS=-O2 --enable-dnscert --enable-ccm --enable-chapoly --enable-ctr --enable-gcm --enable-rdrand --enable-aesni --enable-vici --enable-swanctl --disable-ikev1 --enable-newhope --enable-mgf1 --enable-sha3 --enable-eap-identity --enable-eap-mschapv2 --enable-md4 --enable-pubkey --enable-pkcs11 --enable-openssl
```
### 1.2 Issue free certificate (RSA or ECDSA)

#### 1.2.1 clone [acme.sh](https://github.com/Neilpang/acme.sh) to issue certificate
```
# git clone https://github.com/Neilpang/acme.sh
```

##### [ECDSA certiifcate](https://en.wikipedia.org/wiki/Elliptic_Curve_Digital_Signature_Algorithm) - recommend
```
# ./acme.sh --issue --standalone -d xyz.wuruxu.com --keylength ec-384 --server letsencrypt

```
##### [RSA certificate](https://en.wikipedia.org/wiki/RSA_(cryptosystem))
```
# ./acme.sh --issue --standalone -d xyz.wuruxu.com --keylength 4096 --server letsencrypt
```
#### 1.2.2 install certificates in strongswan
```
# cp ca.cer /etc/ipsec.d/cacerts/
# cp fullchain.cer  /etc/ipsec.d/certs/acme_xyz_server.cert.pem
# cp xyz.wuruxu.com.key  /etc/ipsec.d/private/acme_xyz_ecc.pem
```
###1.3 Configure strongswan
##### 1.3.1 /etc/ipsec.conf
```
# ipsec.conf - strongSwan IPsec configuration file

config setup
  uniqueids=never

conn %default
  keyexchange=ikev2
  left=%defaultroute
  leftauth=pubkey
  leftsubnet=0.0.0.0/0
  leftfirewall=yes
  right=%any
  mobike=yes
  compress=yes
  ike=aes256-sha512-modp4096,aes128-sha512-modp4096,aes256ccm96-sha384-modp2048,aes256-sha256-modp2048,aes128-sha256-modp2048,aes128-sha1-modp2048!
  esp=aes256gcm128,aes128gcm128,aes256ccm128,aes256
  
 conn myvpn
  leftsendcert=always
  leftcert=acme_xyz_server.cert.pem
  leftid=xyz.wuruxu.com
  rightauth=eap-mschapv2
  rightsourceip=10.18.0.0/24
  rightsendcert=never
  eap_identity=%any
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
## 2. Client Configuration
![support platfom](res/images/platform-all.png)      
* [Window 7+](https://wiki.strongswan.org/projects/strongswan/wiki/Windows7#C)
* [iOS & MAC OSX](https://wiki.strongswan.org/projects/strongswan/wiki/AppleClients)
* [Android 4+](https://wiki.strongswan.org/projects/strongswan/wiki/AndroidVPNClient)
* Linux       

#### 1. install strongswan (same options as above)
```
#./configure --prefix=/usr --sysconfdir=/etc --localstatedir=/var CFLAGS=-O2 --enable-dnscert --enable-ccm --enable-chapoly --enable-ctr --enable-gcm --enable-rdrand --enable-aesni --enable-vici --enable-swanctl --disable-ikev1 --enable-newhope --enable-mgf1 --enable-sha3 --enable-eap-identity --enable-eap-mschapv2 --enable-md4 --enable-pubkey --enable-pkcs11 --enable-openssl
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
  ike=aes256-sha512-modp4096,aes128-sha512-modp4096,aes256ccm96-sha384-modp2048,aes256-sha256-modp2048,aes128-sha256-modp2048,aes128-sha1-modp2048!
  esp=aes256gcm128,aes128gcm128,aes256ccm128,aes256

conn ec3
  left=%any
  leftfirewall=yes
  leftauth=eap-mschapv2
  leftsourceip=%config4
  eap_identity=user  #same as /etc/ipsec.secrets
  right=xyz.wuruxu.com
  rightauth=pubkey
  rightid=xyz.wuruxu.com
  rightsubnet=0.0.0.0/0
  auto=add
  
conn local-net
  leftsubnet=192.168.108.0/24
  rightsubnet=192.168.108.0/24
  authby=never
  type=pass
  auto=route

conn local-net2
  leftsubnet=192.168.128.0/24
  rightsubnet=192.168.128.0/24
  authby=never
  type=pass
  auto=route

```
##### 2.3 start up connection ec3
```
# ipsec start
# ipsec up ec3
```
##### NOTES: mtu size issue on linux platform #####
edit /etc/strongswan.d/charon/kernel-netlink.conf to change mtu size from 0 to 1280
```
mtu = 1280
```
