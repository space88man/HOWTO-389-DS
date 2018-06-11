# Installation

## CentOS 7 Installation

Install packages:

```
[root@b9384fa19c2f ~]# yum install 389-ds

[root@b9384fa19c2f ~]# rpm -qa | grep ^389
389-console-1.1.18-1.el7.noarch
389-ds-base-1.3.7.5-21.el7_5.x86_64
389-ds-console-1.2.16-1.el7.noarch
389-admin-console-doc-1.1.12-1.el7.noarch
389-dsgw-1.1.11-5.el7.x86_64
389-admin-1.1.46-1.el7.x86_64
389-admin-console-1.1.12-1.el7.noarch
389-ds-console-doc-1.2.16-1.el7.noarch
389-ds-1.2.2-6.el7.noarch
389-adminutil-1.1.21-2.el7.x86_64
389-ds-base-libs-1.3.7.5-21.el7_5.x86_64


```

Run interactive installation script:
```

[root@b9384fa19c2f ~]# setup-ds-admin.pl

## answer all the questions...
## see setupExample.log

[root@b9384fa19c2f ~]# systemctl list-units | grep dirsrv
dirsrv-admin.service                                   loaded active running   389 Administration Server.
dirsrv@ldap180.service                                 loaded active running   389 Directory Server ldap180.
system-dirsrv.slice                                    loaded active active    system-dirsrv.slice

[root@b9384fa19c2f ~]# ss -apnt
State       Recv-Q Send-Q              Local Address:Port                             Peer Address:Port              
LISTEN      0      0                               *:9830                                        *:*                  
LISTEN      0      0                              :::389                                        :::*                  

## 9830 is a web app, the admin server
## 389 is the first directory server
```

Smoke-test:
```
[root@b9384fa19c2f ~]# ldapsearch -D 'cn=Directory Manager' -W -b "dc=example,dc=com" dn
Enter LDAP Password: 
# extended LDIF
#
# LDAPv3
# base <dc=example,dc=com> with scope subtree
# filter: (objectclass=*)
# requesting: dn 
#

# example.com
dn: dc=example,dc=com

# Directory Administrators, example.com
dn: cn=Directory Administrators,dc=example,dc=com

# Groups, example.com
dn: ou=Groups,dc=example,dc=com

# People, example.com
dn: ou=People,dc=example,dc=com

# Special Users, example.com
dn: ou=Special Users,dc=example,dc=com

# Accounting Managers, Groups, example.com
dn: cn=Accounting Managers,ou=Groups,dc=example,dc=com

# HR Managers, Groups, example.com
dn: cn=HR Managers,ou=Groups,dc=example,dc=com

# QA Managers, Groups, example.com
dn: cn=QA Managers,ou=Groups,dc=example,dc=com

# PD Managers, Groups, example.com
dn: cn=PD Managers,ou=Groups,dc=example,dc=com

# search result
search: 2
result: 0 Success

# numResponses: 10
# numEntries: 9
```

## Client

On another machine you can use the client `389-console`; if you have
used it before and setup TLS, then accessing the new instance won't
work. Clean up previous client configuration, especially TLS:

```
[user@client ~]$ rpm -q 389-console
389-console-1.1.18-5.fc28.noarch

## these preferences may cause problems if you are rebuilding an instance
## but had previously used TLS to connect to the previous one as the
## brand new instance doesn't have TLS configured

[user@client ~]$ cd .389-console/
[user@client .389-console]$ ls
caplugin  cert8.db  cert9.db  Console.1.1.17.Login.preferences  jars  key3.db  key4.db  pkcs11.txt  secmod.db
[user@client .389-console]$ rm -rf *
```

First login:

![389-console Login](../master/images/admin-console.png)

Console:

![389-console](../master/images/console2.png)


## Hashed Passwords

New versions of 389-DS use the `PBKDF2_SHA256` hashed password scheme.
The `userPassword` attr will look like

```
# output by ldapsearch as base64 encoded
userPassword: e1BCS0RGMl9TSEEyNTZ9QUFBSUFFR0JaRUI3aTlKY0xqd04zb1MvKzlSYXNLb1J
 xSk43QnNvYitFT3huY21weVF6QWlhNWRUanJhdm9vMG9iQTF2dXpsYXlMZVpRL292VkdLTEN5b0FC
 UEVPWU83ejBUbW9wTXdEQ3lkZ2RxdzlsSkN5QmVXaUxuSE13VysrY2dvbEl3WE1YNlQ2OFE1SEdsR
 2cvS3h0VXQ3Z0YzbmdlWGhqS2dGbTNtSlFtY1lmTFJmRWExNWM5djJPSGRCaWxkL0FCeEkzVEZEK3
 ZMQmp4UjNOQmFUWVlwU3BUTUo2K3NJL3AvZUh2RmdVRFA4QVNVOVRlK0RSRS9QRlpqNGRycms1LzN
 lRnVaZUNnM0JSblVQZEhvdm9LVFVBVVY3T2NzckxoZGtXZXRXM21FdE5FMjdrdFBtc3h5eTVCWEVJ
 N3hkMmpxazhHdkdLOUpxWWQwWjV2cS82ODZ6SE8zQTVSZ3NnWm4waFd1aXVRZGhCNFR3WlpMNU9Id
 UFkSytXazBHWU4rUHF3RHhyWU9SNkYwWExxOHlBSnhiMjVLTnBjUkFveHJMbzhJVlNUVHRk
 
# userPassword: actual value
{PBKDF2_SHA256}AAAIAOdYlzx5TrqsOw1e8zCLwuTk24yC9\
EmBt/UUPl4RNrfUiiE9ugh7gD3VLonI8ynV6TxHDzX1TVf2w\
UB36aIGmedPne1K8FIjLt5XzxedpVAjx0yM7OXk+UlkIunil\
+GbAF7xQ7tvvJomWUzv0xjFqsBbw4AMDubByAY4Bf48kJazr\
ABGBviam2u3Xz9ced4yqnsNfFONeWCpH+B6Ha81QLdC1cbD0\
7w23qEWpwDJRwEuUpUDt6TJ2SfrPQ0F/OPwrh2gslbpmhVCa\
m7fqsI1wC8Ye2aeVZq61jaQD7cHeDff5NHVIrh0vA+F/sRU8\
6TeVYfNtpd0NWe+8Z6NElf4PuqhtlMsOGXd34TkJg3h4ofHf\
yxvT6CwSvOuoIkUTuVtHvpxiiBHfwD2WTg5ggcfXh9iKz4Ag\
5UPPvPBXbhT9H/L
```

The value is 324 bytes long. It is the PBKDF2 hash of the
password formatted as `iteration count(4) + salt(64) + hash(256)`.
