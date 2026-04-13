---
title: "HPE Primera LDAPS Configuration"
date: 2023-10-30T18:59:18+02:00
draft: false
tags: [hpe, server, lang-english]
---

# HPE Primera LDAPS

How to get it working and issues you might encounter.

## Config

You unfortunately cannot get LDAPS working with SSMC GUI, so you must do the basic config via CLI.

### CLI

Open SSH to your primera/3par and type following commands.

First, we set up necessary LDAPS servers:

`setauthparam -f ldap-server "fqdn.of.dc.local"` < Type this changing the FQDN of DC until you got atleast two.

`setauthparam -f ldap-port 636` < Set LDAP port to use LDAPS 636 or GC port 3269.

`setauthparam -f ldap-type MSAD`< Set LDAP type to MS Active Directory.

`setauthparam -f binding sasl` < Set authentication security to encrypt messages.

`setauthparam -f sasl-mechanism DIGEST-MD5` < Set data encryption layer.

`setauthparam -f ldap-reqcert 1` < Require LDAP servers to have trusted certificate.


Following command will import root CA certificate, so that the appliance can verify your LDAPS server presented certificate. Primera will only need the root, it will trust any subordinate CA signed by the root.

```
setauthparam -f ldap-ssl-cacert: 
-----BEGIN CERTIFICATE-----
your own CA root certificate public key here
-----END CERTIFICATE-----
```

Next we set up few required settings regarding user authentication:

`setauthparam -f account-obj user` < this sets LDAP to search for users.

`setauthparam -f account-name-attr sAMAccountName` < this sets LDAP attribute to compare user submitted account against.

`setauthparam -f memberof-attr memberOf` < This sets LDAP attribute to check for users groups.

`setauthparam -f member-attr member` < This is the magic part which is not mentioned anywhere but in CLI Administration guide. Without this you will get weird errors that account is not found inside the group, even though the lookup is done via the user and memberof attribute.

`setauthparam -f accounts-dn "OU=Company,DC=dc,DC=local"` < Set to correct OU for user lookups, will not work without atleast one OU defined...

`setauthparam -f kerberos-realm DC.LOCAL` < Set to netbios name of your domain.


### SSMC GUI

Login to your SSMC GUI to set up user maps easier than in CLI.

1. Go to Security -> LDAP
2. Select your System -> Actions -> Edit Authorization
3. Press "Add authorizations"
4. From dropdown, select super-map for super rights on system.
5. Type in the DN of the group, eg "CN=primera_super_auth,OU=Company,DC=dc,DC=local"
6. Press OK

### Back to CLI

Test if your account works:

`checkpassword username` < Type the username without domain.
You should receive the following details:

```
PRIMERA-1 cli% checkpassword xxxxx
password:
+ attempting authentication and authorization using system-local data
+ authentication denied: unknown username
+ attempting authentication and authorization using LDAP
+ temporarily setting name-to-address mapping: fqdn.of.dc.local -> 192.168.0.1
+ connecting to LDAP server using URI: ldaps://fqdn.of.dc.local
+ binding to user "xxxxx" with SASL mechanism DIGEST-MD5
+ searching LDAP using:
        search base:    OU=Company,DC=dc,DC=local
        scope:          sub
        filter:         (&(objectClass=user)(sAMAccountName=xxxxx))
        for attributes: memberOf userAccountControl accountExpires lockoutTime
+ search result DN: CN=xxxxx,OU=Company,DC=dc,DC=local
+ search result:    memberOf: CN=primera_super_auth,OU=Company,DC=dc,DC=local
...
...
+ search result:    userAccountControl: 512
+ search result:    accountExpires: 9223372036854775807
+ search result:    lockoutTime: 0
+ mapping rule: super mapped to by "CN=primera_super_auth,OU=Company,DC=dc,DC=local"
+ rule match: super mapped to by "CN=primera_super_auth,OU=Company,DC=dc,DC=local"
user xxxxx is authenticated and authorized
```

## Errors

### Authorization denied: operations error

```
+ attempting authentication and authorization using system-local data
+ authentication denied: unknown username
+ attempting authentication and authorization using LDAP
+ temporarily setting name-to-address mapping: fqdn.of.dc.local -> 192.168.0.1
+ connecting to LDAP server using URI: ldaps://fqdn.of.dc.local
+ binding to user "xxxxx" with SASL mechanism DIGEST-MD5
+ searching LDAP using:
        search base:    DC=dc,DC=local
        scope:          sub
        filter:         (&(objectClass=user)(sAMAccountName=xxxxx))
        for attributes: memberOf userAccountControl accountExpires lockoutTime
+ authorization denied: Operations error
user xxxxx is not authenticated or not authorized
```

Your searchbase DN is missing a sub OU.