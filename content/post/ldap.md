+++
title = "Ldap"
date = "2018-02-07T20:00:20+02:00"
tags = ['ldap']
draft = true
+++

Other Stuff.<!--more-->

slapcat

ldapsearch -x -L -D "cn=admin,dc=example,dc=com" -W -b 'dc=example,dc=com' '(objectclass=*)'
ldapsearch -x -L -D "cn=admin,cn=config" -W -b 'cn=config' '(objectclass=*)'


https://github.com/osixia/docker-openldap

Organisation and Organisation role
OLC
