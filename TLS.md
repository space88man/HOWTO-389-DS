# TLS

## Notes

2018-06-11 — v1.40: EC certs are supported; the `nsSSL3Ciphers` attr of `cn=encryption,cn=config`
is important to get right to support EC and TLSv1.3.

`ldap180` is the name of our test Directory Server instance.

Add your trust anchors to the NSS database:

```
# $DIR is where the admin server or directory server instance are located
# E.g., /etc/dirsrv/admin-serv /etc/dirsrv/slapd-ldap180


# Initialize if necessary and save pin soemwhere
certutil -d $DIR -N # to initialize with new pin

# Smoke-tests
certutil -d $DIR -K # to list keys
certutil -d $DIR -L # to list certs

certutil -A -n root-ca -t CT,, -d $DIR -a -i ~user/root-ca.pem
certutil -A -n intermediate-ca -t CT,, -d $DIR -a -i ~user/intermediate.pem

## generate CSR
### RSA-2048
certutil -R -s 'CN=ldap180.example.com' -o example.req -k rsa -d $DIR -a
### EC prime256v1
certutil -R -s 'CN=ldap180.example.com ECDSA' -o example.req -k ec -q nistp256 -d $DIR -a

## import cert from stdin
certutil -A -n admin-serv -t u,u,u -d $DIR -a
# OR
certutil -A -n ldap180-serv -t u,u,u -d $DIR -a
```

## 389-Console

Verbose mode:

```
## Obtain logging jars
wget http://central.maven.org/maven2/org/slf4j/slf4j-api/1.7.25/slf4j-api-1.7.25.jar
wget http://central.maven.org/maven2/org/slf4j/slf4j-simple/1.7.25/slf4j-simple-1.7.25.jar

VERBOSE=1 CLASSPATH=slf4j-api-1.7.25.jar:slf4j-simple-1.7.25.jar /usr/bin/389-console

# you may be prompted to add the TLS trust anchors...

Java virtual machine used: /usr/share/java-utils/java-wrapper
classpath used: /usr/lib/java/jss4.jar:/usr/share/java/ldapjdk.jar:/usr/share/java/idm-console-base.jar:/usr/share/java/idm-console-mcc.jar:/usr/share/java/idm-console-mcc_en.jar:/usr/share/java/idm-console-nmclf.jar:/usr/share/java/idm-console-nmclf_en.jar:/usr/share/java/389-console_en.jar:slf4j-api-1.7.25.jar:slf4j-simple-1.7.25.jar
main class used: com.netscape.management.client.console.Console
flags used:
options used:
arguments used:
[main] INFO org.mozilla.jss.CryptoManager - CryptoManager: loading JSS library
[main] INFO org.mozilla.jss.CryptoManager - CryptoManager: loaded JSS library from /usr/lib64/jss/libjss4.so
[main] INFO org.mozilla.jss.CryptoManager - CryptoManager: initializing NSS database at /home/user/.389-console/

```

## Admin Server

CA Certs:

![CA Certs](../master/images/cacerts.png)

Admin Server cert:

![Admin Cert](../master/images/adminservcrt.png)




This is a web app fronted by Apache HTTPD; after configuring the trust anchors and certificate
we need to add the NSS passphrase to `/etc/dirsrv/admin-serv/pin.txt` for unattended startup.


```
# pin.txt; where XXXXXXXX is the NSS db password
echo 'internal:XXXXXXXX' > /etc/dirsrv/admin-serv/pin.txt

# the following line in /etc/dirsrv/admin-serv/nss.conf
NSSPassPhraseDialog  file:/etc/dirsrv/admin-serv/pin.txt

# the following lines are important in /etc/dirsrv/admin-serv/console.conf
NSSCipherSuite +ecdh_ecdsa_aes_128_sha,+ecdh_ecdsa_aes_256_sha,+ecdhe_ecdsa_aes_128_gcm_sha_256,+ecdhe_ecdsa_aes_128_sha,+ecdhe_ecdsa_aes_128_sha_256,+ecdhe_ecdsa_aes_256_gcm_sha_384,+ecdhe_ecdsa_aes_256_sha,+ecdhe_ecdsa_aes_256_sha_384,+ecdhe_rsa_aes_128_gcm_sha_256,+ecdhe_rsa_aes_128_sha,+ecdhe_rsa_aes_128_sha_256,+ecdhe_rsa_aes_256_gcm_sha_384,+ecdhe_rsa_aes_256_sha,+ecdhe_rsa_aes_256_sha_384,+ecdh_rsa_aes_128_sha,+ecdh_rsa_aes_256_sha,+rsa_aes_128_gcm_sha_256,+rsa_aes_128_sha,+rsa_aes_256_gcm_sha_384,+rsa_aes_256_sha

NSSProtocol TLSv1.2

```

For added TLSv1.3,

```
# the following lines are important in /etc/dirsrv/admin-serv/console.conf
NSSCipherSuite +aes_128_gcm_sha_256,+chacha20_poly1305_sha_256,+ecdh_ecdsa_aes_128_sha,+ecdh_ecdsa_aes_256_sha,+ecdhe_ecdsa_aes_128_gcm_sha_256,+ecdhe_ecdsa_aes_128_sha,+ecdhe_ecdsa_aes_128_sha_256,+ecdhe_ecdsa_aes_256_gcm_sha_384,+ecdhe_ecdsa_aes_256_sha,+ecdhe_ecdsa_aes_256_sha_384,+ecdhe_rsa_aes_128_gcm_sha_256,+ecdhe_rsa_aes_128_sha,+ecdhe_rsa_aes_128_sha_256,+ecdhe_rsa_aes_256_gcm_sha_384,+ecdhe_rsa_aes_256_sha,+ecdhe_rsa_aes_256_sha_384,+ecdh_rsa_aes_128_sha,+ecdh_rsa_aes_256_sha,+rsa_aes_128_gcm_sha_256,+rsa_aes_128_sha,+rsa_aes_256_gcm_sha_384,+rsa_aes_256_sha

NSSProtocol TLSv1.2,TLSv1.3
```
## Directory Server

By default, the enabled ciphers don't work with TLSv1.3 or EC certificates.
Don't adjust in the the console with
`Encryption->Settings...` as you will lose this setting.

```
dn: cn=encryption,cn=config
changetype: modify
delete: nsSSL3Ciphers

dn: cn=encryption,cn=config
changetype: modify
add: nsSSL3Ciphers
nsSSL3Ciphers: -all,+TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,+TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,+TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA,+TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256,+TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA,+TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA384,+TLS_AES_128_GCM_SHA256,+TLS_AES_256_GCM_SHA384,+TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,+TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,+TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA,+TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA,+TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA256,+TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA384
```

Verify:
```
ldapsearch -D 'cn=Directory Manager' -W -b "cn=config" -o ldif-wrap=no   | grep nssslenabled
nssslenabledciphers: TLS_AES_128_GCM_SHA256::AES-GCM::AEAD::128
nssslenabledciphers: TLS_AES_256_GCM_SHA384::AES-GCM::AEAD::256
nssslenabledciphers: TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256::AES-GCM::AEAD::128
nssslenabledciphers: TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256::AES-GCM::AEAD::128
nssslenabledciphers: TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384::AES-GCM::AEAD::256
nssslenabledciphers: TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384::AES-GCM::AEAD::256
nssslenabledciphers: TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA::AES::SHA1::256
nssslenabledciphers: TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA::AES::SHA1::128
nssslenabledciphers: TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA::AES::SHA1::128
nssslenabledciphers: TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA256::AES::SHA256::128
nssslenabledciphers: TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256::AES::SHA256::128
nssslenabledciphers: TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA::AES::SHA1::256
nssslenabledciphers: TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA384::AES::SHA384::256
nssslenabledciphers: TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA384::AES::SHA384::256

```


```
echo 'Internal (Software) Token:XXXXXXXX' > /etc/dirsrv/slapd-ldap180/pin.txt
systemctl restart dirsrv@ldap180

# ss -apnt
State      Recv-Q Send-Q     Local Address:Port                    Peer Address:Port
LISTEN     0      0                      *:9830                               *:*
LISTEN     0      0                     :::636                               :::*
LISTEN     0      0                     :::389                               :::*

## testing LDAPS :636, STARTTLS
ldapsearch -D "cn=Directory Manager" -W -H ldaps://ldap180.example.com -b "cn=config" cn=RSA
ldapsearch -D "cn=Directory Manager" -W -H ldap://ldap180.example.com -ZZ -b "cn=config" cn=RSA

# RSA, encryption, config
dn: cn=RSA,cn=encryption,cn=config
nsSSLToken: internal (software)
nsSSLPersonalitySSL: ldap-rsa
nsSSLActivation: on
objectClass: top
objectClass: nsEncryptionModule
cn: RSA

```


## TLS Smoke Tests

### Admin Server

TLSv1.2 TLSv1.3 to Apache mod_nss:

```
$ openssl s_client -connect 389ds.example.biz:9830 -tls1_2  -CAfile truststore.pem

<...stuff...>
---
No client certificate CA names sent
Peer signing digest: SHA256
Peer signature type: ECDSA
Server Temp Key: X25519, 253 bits
---
SSL handshake has read 1745 bytes and written 321 bytes
Verification: OK
---
New, TLSv1.2, Cipher is ECDHE-ECDSA-AES128-GCM-SHA256
Server public key is 256 bit
Secure Renegotiation IS supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
SSL-Session:
    Protocol  : TLSv1.2
    Cipher    : ECDHE-ECDSA-AES128-GCM-SHA256
    Session-ID: 0A8440ED85B730591C813418D22180C77B8010B6B369B9BE565D3764473748CC
    Session-ID-ctx: 
    Master-Key: B591798760C35D2EF65B56E55648C1B459E90D3C505F959C861AC4E6E4906D8B730C26C89F7B36C5FC2E2EC7B0C6C97F
    PSK identity: None
    PSK identity hint: None
    SRP username: None
    Start Time: 1547345539
    Timeout   : 7200 (sec)
    Verify return code: 0 (ok)
    Extended master secret: no
---

$ openssl s_client -connect 389ds.example.biz:9830 -tls1_3  -CAfile truststore.pem 


<...stuff...>
1aJEMxqCkjAcBgNVHREEFTATghEzODlkcy5leGFtcGxlLmJpejATBgNVHSUEDDAK
BggrBgEFBQcDATAKBggqhkjOPQQDAgNJADBGAiEA91YGzIumynIGV+xlKU0zJ1T+
W35jNLdFMp4yiS95D+QCIQDg3/5ps4fOHCYSEIyWmnyJVG6JrGv5aOdvyBRi9th7
UQ==
-----END CERTIFICATE-----
subject=DC = biz, DC = example, CN = 389ds.example.bz

issuer=DC = biz, DC = example, CN = Intermediate CA

---
No client certificate CA names sent
Peer signing digest: SHA256
Peer signature type: ECDSA
Server Temp Key: X25519, 253 bits
---
SSL handshake has read 1779 bytes and written 307 bytes
Verification: OK
---
New, TLSv1.3, Cipher is TLS_AES_128_GCM_SHA256
Server public key is 256 bit
Secure Renegotiation IS NOT supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
Early data was not sent
Verify return code: 0 (ok)
---
```

### Directory Server

TLSv1.2 TLSv1.3 to JSS

```
$ openssl s_client -connect 389ds.example.biz:636 -tls1_2  -CAfile truststore.pem

<...stuff...>
---
No client certificate CA names sent
Peer signing digest: SHA256
Peer signature type: ECDSA
Server Temp Key: X25519, 253 bits
---
SSL handshake has read 1775 bytes and written 307 bytes
Verification: OK
---
New, TLSv1.3, Cipher is TLS_AES_128_GCM_SHA256
Server public key is 256 bit
Secure Renegotiation IS NOT supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
Early data was not sent
Verify return code: 0 (ok)
---

$ openssl s_client -connect 389ds.example.biz:636 -tls1_3  -CAfile truststore.pem 

<...stuff...>
---
No client certificate CA names sent
Peer signing digest: SHA256
Peer signature type: ECDSA
Server Temp Key: X25519, 253 bits
---
SSL handshake has read 1740 bytes and written 321 bytes
Verification: OK
---
New, TLSv1.2, Cipher is ECDHE-ECDSA-AES128-GCM-SHA256
Server public key is 256 bit
Secure Renegotiation IS supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
SSL-Session:
    Protocol  : TLSv1.2
    Cipher    : ECDHE-ECDSA-AES128-GCM-SHA256
    Session-ID: 0989F017AA0DDF6C90A0A7C8A563A39AD682FE3BBE051177D8C718D626A6A88F
    Session-ID-ctx: 
    Master-Key: 9864EB49F538F77052441327D08BD761A1E7EC18AD253D9F4FC98A5BFEA01E39370E109C5CA38D09AD660E40C6604A8A
    PSK identity: None
    PSK identity hint: None
    SRP username: None
    Start Time: 1547345707
    Timeout   : 7200 (sec)
    Verify return code: 0 (ok)
    Extended master secret: no
---

#### Using STARTTLS

$ openssl s_client -connect 389ds.example.biz:389 -tls1_3  -CAfile truststore.pem  -starttls ldap

<...stuff...>
UQ==
-----END CERTIFICATE-----
subject=DC = biz, DC = example, CN = 389ds.example.bz

issuer=DC = biz, DC = example, CN = Intermediate CA

---
No client certificate CA names sent
Peer signing digest: SHA256
Peer signature type: ECDSA
Server Temp Key: X25519, 253 bits
---
SSL handshake has read 1871 bytes and written 338 bytes
Verification: OK
---
New, TLSv1.3, Cipher is TLS_AES_128_GCM_SHA256
Server public key is 256 bit
Secure Renegotiation IS NOT supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
Early data was not sent
Verify return code: 0 (ok)
---

$ openssl s_client -connect 389ds.example.biz:389 -tls1_2  -CAfile truststore.pem  -starttls ldap

<...stuff...>
---
No client certificate CA names sent
Peer signing digest: SHA256
Peer signature type: ECDSA
Server Temp Key: X25519, 253 bits
---
SSL handshake has read 1836 bytes and written 352 bytes
Verification: OK
---
New, TLSv1.2, Cipher is ECDHE-ECDSA-AES128-GCM-SHA256
Server public key is 256 bit
Secure Renegotiation IS supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
SSL-Session:
    Protocol  : TLSv1.2
    Cipher    : ECDHE-ECDSA-AES128-GCM-SHA256
    Session-ID: 0989C19096C3478F385A8E52295E8BA9054C62EA8D14FE2D51365E5B88DAF319
    Session-ID-ctx: 
    Master-Key: AF65528F82B3F5ADDE8FBB689B8640E223C0340F0D9CEF4BCAD73945384AB68B8165139A0B33EABD9EF00BFB94439C74
    PSK identity: None
    PSK identity hint: None
    SRP username: None
    Start Time: 1547345802
    Timeout   : 7200 (sec)
    Verify return code: 0 (ok)
    Extended master secret: no
---


```
