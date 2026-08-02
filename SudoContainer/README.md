

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

[root@thuner-srv1 ~]# emacs /etc/openldap/schema/sudoers.ldif

[root@thuner-srv1 ~]# cat /etc/openldap/schema/sudoers.ldif
dn: ou=sudo,dc=cpp,dc=ualberta,dc=ca
objectClass: organizationalUnit
objectClass: top
ou: sudo
description: Default ou for SUDO



[root@thuner-srv1 ~]# ldapadd -x -D cn=ldapadm,dc=cpp,dc=ualberta,dc=ca -W -f /etc/openldap/schema/sudoers.ldif


[root@thuner-srv1 ~]# ldapsearch -x -D "cn=ldapadm,dc=cpp,dc=ualberta,dc=ca" -W -b "dc=cpp,dc=ualberta,dc=ca" "(ou=sudo)"







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
[root@thuner-srv1 ~]# ls
anaconda-ks.cfg  ldap  ldap_backup_2025-10-26  munge.key  openssl_slapd.cnf
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

