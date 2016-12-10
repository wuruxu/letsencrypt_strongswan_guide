# letsencrypt_strongswan_guide

##Requirement
* A domain name: for example **xyz.wuruxu.com**
* VPS: such as [Linode VPS](https://www.linode.com/?r=0bc6a0c838d110075a691b29f2c49d9e90ce2eed)
* Ports 4500/UDP, 500/UDP, 51/UDP and 50/UDP opened in the firewall

##VPN Server Setup
###Build strongswan
openssl >= 1.0.2 is required for add ECP521/ECP384/ECP256 & ECDSA support
```
# apt-get install libssl-dev
```
```
#./configure --prefix=/usr --sysconfdir=/etc --localstatedir=/var CFLAGS=-O2 --enable-dnscert --enable-ccm --enable-chapoly --enable-ctr --enable-gcm --enable-rdrand --enable-aesni --enable-vici --enable-swanctl --disable-ikev1 --enable-newhope --enable-mgf1 --enable-sha3 --enable-eap-identity --enable-eap-mschapv2 --enable-md4 --enable-pubkey --enable-pkcs11 --enable-openssl
```
###Issue free certificate (RSA or ECDSA)

####get tools for issue certificate
[acme.sh](https://github.com/Neilpang/acme.sh)
```
# git clone https://github.com/Neilpang/acme.sh
```

####[ECDSA certiifcate](https://en.wikipedia.org/wiki/Elliptic_Curve_Digital_Signature_Algorithm) recommend
```
# ./acme.sh --issue --standalone -d xyz.wuruxu.com --keylength ec-384

```
####[RSA certificate](https://en.wikipedia.org/wiki/RSA_(cryptosystem))
```
# ./acme.sh --issue --standalone -d xyz.wuruxu.com --keylength 4096
```
