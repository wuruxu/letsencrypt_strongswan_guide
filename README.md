# letsencrypt_strongswan_guide

[![letsencrypt](res/images/letsencrypt-logo-horizontal.png)](https://letsencrypt.org/) ![puls](res/images/add-symbolic.symbolic.png) [![strongswan](res/images/strongswan.png)](https://strongswan.org/)
##Requirement
* A domain name: for example **xyz.wuruxu.com**
* VPS: such as [Linode VPS](https://www.linode.com/?r=0bc6a0c838d110075a691b29f2c49d9e90ce2eed)
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

#### [ECDSA certiifcate](https://en.wikipedia.org/wiki/Elliptic_Curve_Digital_Signature_Algorithm) - recommend
```
# ./acme.sh --issue --standalone -d xyz.wuruxu.com --keylength ec-384

```
#### [RSA certificate](https://en.wikipedia.org/wiki/RSA_(cryptosystem))
```
# ./acme.sh --issue --standalone -d xyz.wuruxu.com --keylength 4096
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
: RSA vpn.mydomain.com.key
: ECDSA acme_ec6_ecc.pem
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
