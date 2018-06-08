# Plugins

This is a useful set of plugins if you want LDAP to
behave as a unix identity provider ("looks like `/etc/passwd`").
With these plugins, it should behave well with `pam_ldap` or `pam_sssd`.

## DNA — Distributed Numeric Assignment

Used to generate auto-incrementing POSIX uids, gids for user accounts.
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

Used to auto-magically create a user private group, a `posixGroup` object with the same
name as the uid, just like how `useradd` works with `USERGROUPS_ENAB yes`.

Create the following template:

```
# Templates, example.com
dn: ou=Templates,dc=example,dc=com
ou: Templates
objectClass: top
objectClass: organizationalunit

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

The origin `posixAccount` object has the following attributes:

```
# vagrant, People, example.com
dn: uid=vagrant,ou=People,dc=example,dc=com
objectClass: mepOriginEntry
mepManagedEntry: cn=vagrant,ou=groups,dc=example,dc=com
```

The managed entry `posixGroup` has the following attributes:

```
# vagrant, Groups, example.com
dn: cn=vagrant,ou=Groups,dc=example,dc=com
objectClass: posixGroup
objectClass: mepManagedEntry
objectClass: top
cn: vagrant Group
cn: vagrant
gidNumber: 5001
memberUid: vagrant
mepManagedBy: uid=vagrant,ou=People,dc=example,dc=com
```

Notice the `mepManagedEntry` and `mepManagedBy` attributes on origin, and managed entry.

## memberOf Plugin

This plugin is used to manage the set of `memberOf` attributes in a user object.

When you create a group object and add a user object(uid) as a `member` attribute,
a reverse `memberOf` attribute is created in the corresponding user.
This is a back-pointer to the corresponding group.

This convenience plugin allows us to get the list of groups a user belongs to without having to do
a complicated search.

Note: For posixGroup objects(actually groupOfUniqueNames proxy object is used) it is more natural to use `uniquemember` instead of `member`.

Configuration, let's configure the plugin in the DIT:

```
# add this attribute
# MemberOf Plugin, plugins, config

dn: cn=MemberOf Plugin,cn=plugins,cn=config
changetype: modify
add: nsslapd-pluginConfigArea
nsslapd-pluginConfigArea: cn=MemberOf Plugin Configuration,ou=Templates,dc=example,dc=com

# configuration using a separate object, instead of the plugin object
# add this object to the tree
# the ou=Templates came from the MEP plugin

dn: cn=MemberOf Plugin Configuration,ou=Templates,dc=example,dc=com
objectClass: top
objectClass: extensibleObject
cn: MemberOf Plugin Configuration
memberofgroupattr: uniquemember
memberofattr: memberOf
```


Results:

```
#### Create a groupOfUniqueNames
# testgrp, vagrant, Groups, example.com
dn: cn=testgrp,ou=Groups,dc=example,dc=com
gidNumber: 3000
description: Test Group for memberOf plugin
objectClass: groupofuniquenames
objectClass: top
objectClass: posixgroup
cn: testgrp
uniqueMember: uid=vagrant,ou=People,dc=example,dc=com

#### look what happens to user vagrant
# vagrant, People, example.com
dn: uid=vagrant,ou=People,dc=example,dc=com
gidNumber: 5001
uidNumber: 5001
memberOf: cn=testgrp,ou=Groups,dc=example,dc=com
```
