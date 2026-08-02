# LDAP support for centralized sudo policy management

Each compute node maintained its own local sudoers configuration:

```bash
Administrator
        │
        ├── Edit /etc/sudoers on node 1
        ├── Edit /etc/sudoers on node 2
        ├── Edit /etc/sudoers on node 3
        └── ...

```

This approach does not scale well because every policy modification must be propagated manually to every machine, increasing administrative effort and the risk of configuration drift. By extending the LDAP schema, sudo authorization can instead be managed through the directory service:

```bash
Administrator
        │
        ▼
OpenLDAP
(cn=config + sudo schema)
        │
        ▼
ou=SUDOers
        │
        ├── sudoRole
        ├── sudoRole
        ├── sudoRole
        └── ...
        │
        ▼
Compute nodes retrieve policies through LDAP
```

The LDAP server becomes the authoritative repository for sudo authorization, while the compute nodes act as LDAP clients that retrieve policy information at runtime.

## Extending the LDAP directory schema

To centralize administrative privilege management across the cluster, the OpenLDAP directory was extended with the sudo LDAP schema. This schema introduces the sudoRole object class together with the associated attributes (`sudoUser`, `sudoHost`, `sudoCommand`, `sudoRunAsUser`, `sudoOption`, etc.) required by the sudo LDAP backend.

```bash
[root@thuner-srv1 ~]# cp /usr/share/doc/sudo-1.8.23/schema.OpenLDAP /etc/openldap/schema/sudo.schema
[root@thuner-srv1 ~]# emacs /etc/openldap/schema/sudo.ldif
[root@thuner-srv1 ~]# cat /etc/openldap/schema/sudo.ldif
dn: cn=sudo,cn=schema,cn=config
objectClass: olcSchemaConfig
cn: sudo
olcAttributeTypes: ( 1.3.6.1.4.1.15953.9.1.1 NAME 'sudoUser' DESC 'User(s) who may run sudo' EQUALITY caseExactIA5Match SUBSTR caseExactIA5SubstringsMatch SYNTAX 1.3.6.1.4.1.1466.115.121.1.26 )
olcAttributeTypes: ( 1.3.6.1.4.1.15953.9.1.2 NAME 'sudoHost' DESC 'Host(s) who may run sudo' EQUALITY caseExactIA5Match SUBSTR caseExactIA5SubstringsMatch SYNTAX 1.3.6.1.4.1.1466.115.121.1.26 )
olcAttributeTypes: ( 1.3.6.1.4.1.15953.9.1.3 NAME 'sudoCommand' DESC 'Command(s) to be executed by sudo' EQUALITY caseExactIA5Match SYNTAX 1.3.6.1.4.1.1466.115.121.1.26 )
olcAttributeTypes: ( 1.3.6.1.4.1.15953.9.1.4 NAME 'sudoRunAs' DESC 'User(s) impersonated by sudo (deprecated)' EQUALITY caseExactIA5Match SYNTAX 1.3.6.1.4.1.1466.115.121.1.26 )
olcAttributeTypes: ( 1.3.6.1.4.1.15953.9.1.5 NAME 'sudoOption' DESC 'Options(s) followed by sudo' EQUALITY caseExactIA5Match SYNTAX 1.3.6.1.4.1.1466.115.121.1.26 )
olcAttributeTypes: ( 1.3.6.1.4.1.15953.9.1.6 NAME 'sudoRunAsUser' DESC 'User(s) impersonated by sudo' EQUALITY caseExactIA5Match SYNTAX 1.3.6.1.4.1.1466.115.121.1.26 )
olcAttributeTypes: ( 1.3.6.1.4.1.15953.9.1.7 NAME 'sudoRunAsGroup' DESC 'Group(s) impersonated by sudo' EQUALITY caseExactIA5Match SYNTAX 1.3.6.1.4.1.1466.115.121.1.26 )
olcAttributeTypes: ( 1.3.6.1.4.1.15953.9.1.8 NAME 'sudoNotBefore' DESC 'Start of time interval for which the entry is valid' EQUALITY generalizedTimeMatch ORDERING generalizedTimeOrderingMatch SYNTAX 1.3.6.1.4.1.1466.115.121.1.24 )
olcAttributeTypes: ( 1.3.6.1.4.1.15953.9.1.9 NAME 'sudoNotAfter' DESC 'End of time interval for which the entry is valid' EQUALITY generalizedTimeMatch ORDERING generalizedTimeOrderingMatch SYNTAX 1.3.6.1.4.1.1466.115.121.1.24 )
olcAttributeTypes: ( 1.3.6.1.4.1.15953.9.1.10 NAME 'sudoOrder' DESC 'an integer to order the sudoRole entries' EQUALITY integerMatch ORDERING integerOrderingMatch SYNTAX 1.3.6.1.4.1.1466.115.121.1.27 )
olcObjectClasses: ( 1.3.6.1.4.1.15953.9.2.1 NAME 'sudoRole' SUP top STRUCTURAL DESC 'Sudoer Entries' MUST ( cn ) MAY ( sudoUser $ sudoHost $ sudoCommand $ sudoRunAs $ sudoRunAsUser $ sudoRunAsGroup $ sudoOption $ sudoOrder $ sudoNotBefore $ sudoNotAfter $ description ) )
[root@thuner-srv1 ~]# chown ldap:ldap /etc/openldap/schema/sudo.ldif
[root@thuner-srv1 ~]# ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/sudo.ldif
```

The schema definition was installed as an LDIF file, assigned the appropriate ownership, and imported into the dynamic OpenLDAP configuration (`cn=config`) using the local privileged LDAP socket (`ldapi:///`) with SASL EXTERNAL authentication. Once loaded, the LDAP server became capable of storing and serving centralized sudo policy entries.

> **Security objective.** Extend the LDAP directory to support centralized authorization policies. Eliminate the dependency on independently maintained local `/etc/sudoers` files. Provide the schema required for LDAP-based sudo integration.

## The LDAP-sudo container

After extending the LDAP schema with support for sudoRole objects, a dedicated organizational unit (`ou=sudo`) is created within the directory information tree (DIT). This organizational unit serves as the container for all LDAP-based sudo policy entries.

```bash
[root@thuner-srv1 ~]# emacs /etc/openldap/schema/sudoers.ldif
[root@thuner-srv1 ~]# cat /etc/openldap/schema/sudoers.ldif
dn: ou=sudo,dc=cpp,dc=ualberta,dc=ca
objectClass: organizationalUnit
objectClass: top
ou: sudo
description: Default ou for SUDO
[root@thuner-srv1 ~]# ldapadd -x -D cn=ldapadm,dc=cpp,dc=ualberta,dc=ca -W -f /etc/openldap/schema/sudoers.ldif
```

The container is defined as an `organizationalUnit` object beneath the directory suffix (`dc=cpp,dc=ualberta,dc=ca`) and imported into the LDAP directory using the administrative account (`cn=ldapadm`) through a standard LDAP bind. This organizational unit does not contain authorization rules by itself. Instead, it provides the namespace in which `sudoRole` entries can subsequently be created and managed.

> **Security objective.** Establish a dedicated location for centralized sudo authorization policies. Separate privilege-management objects from user and group entries. Provide the parent container required for LDAP-based sudo administration,

The previous step extended the LDAP server so that it understood the `sudoRole` object class. This step creates the directory location where instances of that object class will reside. Conceptually, the two steps are distinct:

```bash
Step 1
-------
Extend the LDAP schema
↓
The directory learns what a sudoRole object is.

Step 2
-------
Create ou=sudo
↓
The directory now has a location where sudoRole objects can be stored.

Step 3
-------
Create sudoRole entries
↓
Actual sudo authorization policies are defined.
```

The resulting DIT becomes:

```bash
dc=cpp,dc=ualberta,dc=ca
├── ou=People
├── ou=Groups
├── ou=Hosts
└── ou=sudo
    ├── cn=ClusterAdmins
    ├── cn=StorageAdmins
    └── cn=Operators
```

Each child of `ou=sudo` is a `sudoRole` object that defines which users may execute which commands on which hosts.


## Deploy a centralized default sudo policy in LDAP

The previous sections established the infrastructure required for centralized privilege management:

* The LDAP schema was extended with support for `sudoRole` objects.
* A dedicated organizational unit (`ou=sudo`) was created to store sudo policies.

This step populates that organizational unit with its first policy object:

```bash
dc=cpp,dc=ualberta,dc=ca
└── ou=sudo
    └── cn=defaults
        ├── sudoOption: env_reset
        ├── sudoOption: mail_badpass
        └── sudoOption: secure_path=...
```
Unlike the organizational unit, which merely acts as a container, the `cn=defaults` entry contains configuration data that is interpreted by LDAP-enabled sudo clients during policy evaluation. Following the creation of the `ou=sudo` organizational unit, a `sudoRole` object (`cn=defaults`) is created to define the default sudo configuration distributed through the LDAP directory.

```bash
[root@thuner-srv1 ~]# emacs /etc/openldap/schema/sudoconf.ldif
[root@thuner-srv1 ~]# cat /etc/openldap/schema/sudoconf.ldif 
dn: cn=defaults,ou=sudo,dc=cpp,dc=ualberta,dc=ca
objectClass: sudoRole
objectClass: top
cn: defaults
sudoOption: env_reset
sudoOption: mail_badpass
sudoOption: secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin
#sudoOrder: 1
[root@thuner-srv1 ~]# chown ldap:ldap /etc/openldap/schema/sudoconf.ldif
[root@thuner-srv1 ~]# ldapadd -x -D "cn=ldapadm,dc=cpp,dc=ualberta,dc=ca" -W -f /etc/openldap/schema/sudoconf.ldif
Enter LDAP Password: 
adding new entry "cn=defaults,ou=sudo,dc=cpp,dc=ualberta,dc=ca"
[root@thuner-srv1 ~]# ldapsearch -x -D "cn=ldapadm,dc=cpp,dc=ualberta,dc=ca" -W -b "ou=sudo,dc=cpp,dc=ualberta,dc=ca" "(cn=defaults)"
Enter LDAP Password: 
# extended LDIF
#
# LDAPv3
# base <ou=sudo,dc=cpp,dc=ualberta,dc=ca> with scope subtree
# filter: (cn=defaults)
# requesting: ALL
#

# defaults, sudo, cpp.ualberta.ca
dn: cn=defaults,ou=sudo,dc=cpp,dc=ualberta,dc=ca
objectClass: sudoRole
objectClass: top
cn: defaults
sudoOption: env_reset
sudoOption: mail_badpass
sudoOption: secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbi
 n:/bin:/snap/bin

# search result
search: 2
result: 0 Success

# numResponses: 2
# numEntries: 1
```

The entry specifies global `sudoOption` directives, including environment sanitization (`env_reset`), email notification on authentication failures (`mail_badpass`), and a controlled execution path (`secure_path`). These options define the baseline behavior of the sudo command for LDAP-enabled clients independently of host-local configuration files. The policy is imported into the directory using the administrative LDAP account (`cn=ldapadm`) and subsequently verified using an LDAP search operation to confirm that the entry had been successfully stored and was retrievable from the directory.

> **Security objective.** Establish a centralized baseline sudo configuration. Enforce consistent sudo behavior across all LDAP-integrated compute nodes. Verify successful deployment of the policy within the LDAP directory.