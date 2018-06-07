# Plugins

This is a useful set of plugins if you want LDAP to
behave as a unix identity provider ("looks like `/etc/passwd`").

## DNA — Distributed Numeric Assignment

Used to auto-generate POSIX uids, gids for user accounts.
Enable the DNA plugin, add the following LDIF, and restart the server.

When you add a user, set the uid/gid to 99999/99999 and they will be replaced
by the auto-generated ones.

```
[root@b9384fa19c2f ldap]# cat dna.ldif 
# uids
dn: cn=UID numbers,cn=Distributed Numeric Assignment Plugin,cn=plugins,cn=config
objectClass: top
objectClass: extensibleObject
cn: UID numbers
dnatype: uidNumber
dnamagicregen: 99999
dnafilter: (objectclass=posixAccount)
dnascope: dc=example,dc=com
dnanextvalue: 5000

# gids
dn: cn=GID numbers,cn=Distributed Numeric Assignment Plugin,cn=plugins,cn=config
objectClass: top
objectClass: extensibleObject
cn: GID numbers
dnatype: gidNumber
dnamagicregen: 99999
dnafilter: (|(objectclass=posixAccount)(objectclass=posixGroup))
dnascope: dc=example,dc=com
dnanextvalue: 5000
```

## MEP — Managed Entries

Used to auto-magically create a user private group, a 'posixGroup` object, like how
`useradd` works with `USERGROUPS_ENAB yes`.

Create the following template:

```
dn: cn=Posix User-Group Template,ou=Templates,dc=example,dc=com
objectclass: mepTemplateEntry
cn: Posix User-Group Template
mepRDNAttr: cn
mepStaticAttr: objectclass: posixGroup
mepMappedAttr: cn: $uid Group
mepMappedAttr: gidNumber: $gidNumber
mepMappedAttr: memberUid: $uid
```

...and... configure the plugin:

```
dn: cn=Posix User-Group,cn=Managed Entries,cn=plugins,cn=config
objectclass: extensibleObject
cn: Posix User-Group
originScope: ou=people,dc=example,dc=com
originFilter: objectclass=posixAccount
managedBase: ou=groups,dc=example,dc=com
managedTemplate: cn=Posix User-Group Template,ou=Templates,dc=example,dc=com
```


