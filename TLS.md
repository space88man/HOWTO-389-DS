# TLS

## Notes

**Directory server instance does not work with EC certificates**

Add your trust anchors to the NSS database:

```
certutil -d $DIR -K # to list keys
certutil -d $DIR -L # to list certs

# $DIR is where the admin server or directory server instance are located
# E.g., /etc/dirsrv/admin-serv /etc/dirsrv/slapd-ldap180

certutil -A -n root-ca -t CT,, -d $DIR -a -i ~user/root-ca.pem
certutil -A -n intermediate-ca -t CT,, -d $DIR -a -i ~user/intermediate.pem 

## generate CSR
### RSA-2048
certutil -R -s 'CN=ldap180.example.com' -o admin.req -k rsa -d $DIR -a
### EC prime256v1
certutil -R -s 'CN=ldap180.example.com ECDSA' -o admin.req -k ec -q nistp256 -d $DIR -a

## import cert from stdin
certutil -A -n admin-serv -t u,u,u -d $DIR -a 
# OR
certutil -A -n ldap180-serv -t u,u,u -d $DIR -a 
```

## Admin Server

CA Certs:

![CA Certs](../master/images/cacerts.png)

Admin Server cert:

![Admin Cert](../master/images/adminservcrt.png)




This is a web app fronted by Apache HTTPD; after configuring the trust anchors and certificate
we need to add the NSS passphrase to `/etc/dirsrv/admin-serv/pin.txt` for unattended startup.


```
# the following line in /etc/dirsrv/admin-serv/nss.conf
NSSPassPhraseDialog  file:/etc/dirsrv/admin-serv/pin.txt

# the following lines are important in /etc/dirsrv/admin-serv/console.conf
NSSCipherSuite +ecdh_ecdsa_aes_128_sha,+ecdh_ecdsa_aes_256_sha,+ecdhe_ecdsa_aes_128_gcm_sha_256,+ecdhe_ecdsa_aes_128_sha,+ecdhe_ecdsa_aes_128_sha_256,+ecdhe_ecdsa_aes_256_gcm_sha_384,+ecdhe_ecdsa_aes_256_sha,+ecdhe_ecdsa_aes_256_sha_384,+ecdhe_rsa_aes_128_gcm_sha_256,+ecdhe_rsa_aes_128_sha,+ecdhe_rsa_aes_128_sha_256,+ecdhe_rsa_aes_256_gcm_sha_384,+ecdhe_rsa_aes_256_sha,+ecdhe_rsa_aes_256_sha_384,+ecdh_rsa_aes_128_sha,+ecdh_rsa_aes_256_sha,+rsa_aes_128_gcm_sha_256,+rsa_aes_128_sha,+rsa_aes_256_gcm_sha_384,+rsa_aes_256_sha

NSSProtocol TLSv1.2

```

## Directory Server

**Use only RSA certificates**

```
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
