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
