# Free Overleaf Ldap Implementation

This repo contains an improved, free ldap authentication and authorisation 
for sharelatex/[overleaf](https://github.com/overleaf/overleaf) community 
edition. Currently this repo uses sharelatex:latest.

The inital idea for this implementation was taken from 
[worksasintended](https://github.com/worksasintended).


### Limitations:

This implementation uses *no* ldap bind user - it tries to bind to the ldap (using ldapts) with 
the uid and credentials of the user which tries to login.


Only valid LDAP users or email users registered by an admin can login. 
This module authenticates against the local DB if `ALLOW_EMAIL_LOGIN` is set to `true` if this fails
it tries to authenticate against the specified LDAP server. 

*Note:*
- LDAP Users can not change their password for the ldap username login. They have to change it at the ldap server.
- LDAP Users can reset their local db password. Then they can decide if they login with either their ldap user and password or with their email and local db password.
- Users can not change their email. The email address is taken from the ldap server (mail) field. (or by invitation through an admin).
  This ldap mail field has to contain a valid mail address. Firstname and lastname are taken from the fields "givenName" and "sn". 
  If you want to use different fields change the code in AuthenticationManager.js lines 297-299.
- Admins can invite non ldap users directly (via email). Additionally (``link sharing`` of projects is possible).

*Important:*
Sharelatex/Overleaf uses the email address to identify users: If you change the email in the LDAP you have to update the corresponding field 
in the mongo db.

```
docker exec -it mongo /bin/bash
mongo 
use sharelatex
db.users.find({email:"EMAIL"}).pretty()
db.users.update({email : OLDEMAIL},{$set: { email : NEWEMAIL}});
```

## Configuration

### Domain Configuration

Edit the [.env](.env) file

```
MYDOMAIN=example.com
MYMAIL=email@example.com
MYDATA=/data
```

*MYDATA* is the location (mount-point) for all data and will hold several directories:
- mongo_data: Mongo DB
- redis_data: Redis dump.rdb
- sharelatex: all projects, tmp files, user files templates and ...
- letsencrypt: https certificates

*MYDOMAIN* is the FQDN for sharelatex and traefik (letsencrypt) or certbot  <br/>
*MYDOMAIN*:8443 Traefik Dashboard (docker-compose-traefik.yml) - Login uses traefik/user.htpasswd : user:admin pass:adminPass change this (e.g. generate a password with htpasswd)
*MYMAIL* is the admin mailaddress

```
LOGIN_TEXT=username
COLLAB_TEXT=Direct share with collaborators is enabled only for activated users!
ADMIN_IS_SYSADMIN=false
```
*LOGIN_TEXT* : displayed instead of email-adress field (login.pug) <br/>
*COLLAB_TEXT* : displayed for email invitation (share.pug)<br/>
*ADMIN_IS_SYSADMIN* : false or true (if ``false`` isAdmin group is allowed to add users to sharelatex and post messages. if ``true`` isAdmin group is allowed to logout other users / set maintenance mode)


### LDAP Configuration

Edit [docker-compose.treafik.yml](docker-compose.traefik.yml) or [docker-compose.certbot.yml](docker-compose.certbot.yml) to fit your local setup. 



```
LDAP_SERVER: ldaps://LDAPSERVER:636
LDAP_BASE: dc=DOMAIN,dc=TLD
LDAP_BINDDN: ou=someunit,ou=people,dc=DOMAIN,dc=TLS
# By default tries to bind directly with the ldap user - this user has to be in the LDAP GROUP
# you have to set a group filter a minimal groupfilter would be: '(objectClass=person)'
LDAP_GROUP_FILTER: '(memberof=GROUPNAME,ou=groups,dc=DOMAIN,dc=TLD)'

# If user is in ADMIN_GROUP on user creation (first login) isAdmin is set to true. 
# Admin Users can invite external (non ldap) users. This feature makes only sense 
# when ALLOW_EMAIL_LOGIN is set to 'true'. Additionally admins can send 
# system wide messages.
#LDAP_ADMIN_GROUP_FILTER: '(memberof=cn=ADMINGROUPNAME,ou=groups,dc=DOMAIN,dc=TLD)'
ALLOW_EMAIL_LOGIN: 'false'

# All users in the LDAP_GROUP_FILTER are loaded from the ldap server into contacts.
LDAP_CONTACTS: 'false'
```

### LDAP Contacts 

If you enable LDAP_CONTACTS, then all users in LDAP_GROUP_FILTER are loaded from the ldap server into the contacts. 
At the moment this happens every time you click on "Share" within a project.
The user search happens without bind - so if your LDAP needs a bind you can adapt this in the 
function `getLdapContacts()` in ContactsController.js (line 92) 
if you want to enable this function set:
```
LDAP_CONTACTS: 'true'
```

### Sharelatex Configuration

Edit SHARELATEX_ environment variables in [docker-compose.traefik.yml](docker-compose.traefik.yml) or [docker-compose.certbot.yml](docker-compose.certbot.yml) to fit your local setup 
(e.g. proper SMTP server, Header, Footer, App Name,...). See https://github.com/overleaf/overleaf/wiki/Quick-Start-Guide for more details.

## Installation, Usage and Inital startup

Install the docker engine: https://docs.docker.com/engine/install/

Install docker-compose:

(if you need pip: apt install python3-pip)

```
pip install docker-compose
```


use the command 
```
make
```
to generate the ldap-overleaf-sl docker image.

use the command
```
docker network create web
```
to create a network for the docker instances.


## Startup 

There are 2 different ways of starting either using Traefik or using Certbot. Adapt the one you want to use.

### Using Traefik

Then start docker containers (with loadbalancer):
``` 
export NUMINSTANCES=1
docker-compose -f docker-compose.traefik.yml up -d --scale sharelatex=$NUMINSTANCES
```

### Using Certbot 
Enable line 65/66 and 69/70 in ldapoverleaf-sl/Dockerfile and ``make`` again.

``` 
docker-compose -f docker-compose.certbot.yml up -d 
```

