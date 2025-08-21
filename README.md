# ansible_ldap_test
Данный репозиторий был создан специально для тестового задания. 

В ходе анализа технического задания выделил следующие инструменты для работы:
* Ansible (требование технического задания)
* QEMU-KVM + virsh (для создания vm, на которой будет разворачиваться OpenLDAP)
* Пакет slapd и часть утилит, входящих в него

В первую очередь создаем и подымаем vm с ОС Ubuntu. После этого прокидываем свой ssh ключ под рута на созданную vm. Далее переходим к нашему мастеру (устройству, с которго будет запускаться плейбук) и клоинруем/качаем данный репозиторий. Переходим в диру с репозиторием и указываем в файле hosts.txt ноду/ноды (в моем случае это ранее поднятая vm). В принципе, этого достаточно для запуска основного плейбука. Запуск происходит из корневой директории репозитория следующим образом:
```
ansible-playbook playbook.yml -i hosts.txt
```
Коротко о том, что будет происходить:
1) Обновится список доступных пакетов из репозиториев, которые указаны в файле /etc/apt/sources.list (apt update):
```BASH
included: /home/local/Documents/ansible/OpenLDAP/tasks/installing_ldap.yml for ubuntu1

TASK [OpenLDAP : Update apt cache] 
```
2) Произойдет проверка на наличие пакетов splad, ldap-utils, python3-ldap на устройстве. Если пакетов нет на ноде, то происводится их установка через пакетный менеджер apt.
```BASH
TASK [OpenLDAP : Install OpenLDAP] 
```
3) Сервис splad поднимется и добавится в автозапуск (systemctl enable):
```BASH
TASK [OpenLDAP : Ensure slapd is running and enabled] 
```
5) Готовый конфиг OpenLDAP отправится на ноду/ноды, заменит текущий:
```BASH
included: /home/local/Documents/ansible/OpenLDAP/tasks/configure_ldap.yml for ubuntu1

TASK [OpenLDAP : Replace config] 
```
6) Сервис перезагрузится:
```BASH
TASK [OpenLDAP : Restart slapd] 
```
7) Создается корневая запись:
```BASH
TASK [OpenLDAP : Create root base]
```
8) Создается юнит для пользователей и они добавляются в систему:
```BASH
included: /home/local/Documents/ansible/OpenLDAP/tasks/adding_users.yml for ubuntu1

TASK [OpenLDAP : Create organizational unit for users]

TASK [OpenLDAP : Create LDAP users]
```
8) Создается юнит для групп и они добавляются в систему:
```BASH
included: /home/local/Documents/ansible/OpenLDAP/tasks/adding_groups.yml for ubuntu1

TASK [OpenLDAP : Create organizational unit for groups]

TASK [OpenLDAP : Create groups]
```

Основные правки в конфиге:
* olcSuffix: dc=example,dc=com (изначально стоит nodomain)
* olcRootDN: cn=admin,dc=example,dc=com
* olcRootPW: {SSHA}J3s1NeRgOzXocPinAkRRAzNHUmJHhr0z (тут для смены пароля использовал утилиту slappasswd и перевод в base64)

---

Тесты, которые я проводил:
1) Приминение конфига:
```BASH
ldapsearch -Q -Y EXTERNAL -H ldapi:/// -b cn=config "(olcDatabase={1}mdb)" olcSuffix olcRootDN olcRootPW
```
ожидаемый результат:
```BASH
# extended LDIF
#
# LDAPv3
# base <cn=config> with scope subtree
# filter: (olcDatabase={1}mdb)
# requesting: olcSuffix olcRootDN olcRootPW 
#

# {1}mdb, config
dn: olcDatabase={1}mdb,cn=config
olcSuffix: dc=example,dc=com
olcRootDN: cn=admin,dc=example,dc=com
olcRootPW: {SSHA}J3s1NeRgOzXocPinAkRRAzNHUmJHhr0z

# search result
search: 2
result: 0 Success

# numResponses: 2
# numEntries: 1
```
2) Админский доступ:
```BASH
ldapwhoami -x -H ldap:/// -D "cn=admin,dc=example,dc=com" -w "changeme"
```
ожидаемый результат:
```BASH
dn:cn=admin,dc=example,dc=com
```
3) Наличие организации:
```BASH
ldapsearch -x -b "dc=example,dc=com" -s base "(objectClass=organization)"
```
ожидаемый результат:
```BASH
# extended LDIF
#
# LDAPv3
# base <dc=example,dc=com> with scope baseObject
# filter: (objectClass=organization)
# requesting: ALL
#

# example.com
dn: dc=example,dc=com
objectClass: top
objectClass: dcObject
objectClass: organization
o: Example Organization
dc: example

# search result
search: 2
result: 0 Success

# numResponses: 2
# numEntries: 1
```
4) Добавление пользователей:
```BASH
ldapsearch -x -b "ou=users,dc=example,dc=com" "(objectClass=inetOrgPerson)"
```
ожидаемый результат:
```BASH
# extended LDIF
#
# LDAPv3
# base <ou=users,dc=example,dc=com> with scope subtree
# filter: (objectClass=inetOrgPerson)
# requesting: ALL
#

# user1, users, example.com
dn: uid=user1,ou=users,dc=example,dc=com
uid: user1
cn: UserN1 UserS1
givenName: UserN1
sn: UserS1
mail: user1@example.com
uidNumber: 10001
gidNumber: 10001
homeDirectory: /home/user1
loginShell: /bin/bash
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: top

# user2, users, example.com
dn: uid=user2,ou=users,dc=example,dc=com
uid: user2
cn: UserN2 UserS2
givenName: UserN2
sn: UserS2
mail: user2@example.com
uidNumber: 10002
gidNumber: 10002
homeDirectory: /home/user2
loginShell: /bin/bash
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: top

# search result
search: 2
result: 0 Success

# numResponses: 3
# numEntries: 2
```
5) Добавление групп:
```BASH
ldapsearch -x -b "ou=groups,dc=example,dc=com" "(objectClass=groupOfNames)"
```
ожидаемый результат:
```BASH
# extended LDIF
#
# LDAPv3
# base <ou=groups,dc=example,dc=com> with scope subtree
# filter: (objectClass=groupOfNames)
# requesting: ALL
#

# group1, groups, example.com
dn: cn=group1,ou=groups,dc=example,dc=com
cn: group1
member: uid=user1,ou=users,dc=example,dc=com
description: First group
objectClass: groupOfNames
objectClass: top

# group2, groups, example.com
dn: cn=group2,ou=groups,dc=example,dc=com
cn: group2
member: uid=user2,ou=users,dc=example,dc=com
description: Second group
objectClass: groupOfNames
objectClass: top

# search result
search: 2
result: 0 Success

# numResponses: 3
# numEntries: 2
```

Полный вывод запуска плейбука на мастере:
* При первом запуске:
```BASH
local@sher-pc:~/Documents/ansible$ ansible-playbook playbook.yml -i hosts.txt 

PLAY [Install OpenLDAP] **********************************************************************************************

TASK [Gathering Facts] ***********************************************************************************************
ok: [ubuntu1]

TASK [OpenLDAP : include_tasks] **************************************************************************************
included: /home/local/Documents/ansible/OpenLDAP/tasks/installing_ldap.yml for ubuntu1

TASK [OpenLDAP : Update apt cache] ***********************************************************************************
changed: [ubuntu1]

TASK [OpenLDAP : Install OpenLDAP] ***********************************************************************************
changed: [ubuntu1]

TASK [OpenLDAP : Ensure slapd is running and enabled] ****************************************************************
changed: [ubuntu1]

TASK [OpenLDAP : include_tasks] **************************************************************************************
included: /home/local/Documents/ansible/OpenLDAP/tasks/configure_ldap.yml for ubuntu1

TASK [OpenLDAP : Replace config] *************************************************************************************
changed: [ubuntu1]

TASK [OpenLDAP : Restart slapd] **************************************************************************************
changed: [ubuntu1]

TASK [OpenLDAP : Create root base] ***********************************************************************************
changed: [ubuntu1]

TASK [OpenLDAP : include_tasks] **************************************************************************************
included: /home/local/Documents/ansible/OpenLDAP/tasks/adding_users.yml for ubuntu1

TASK [OpenLDAP : Create organizational unit for users] ***************************************************************
changed: [ubuntu1]

TASK [OpenLDAP : Create LDAP users] **********************************************************************************
changed: [ubuntu1] => (item=user1)
changed: [ubuntu1] => (item=user2)

TASK [OpenLDAP : include_tasks] **************************************************************************************
included: /home/local/Documents/ansible/OpenLDAP/tasks/adding_groups.yml for ubuntu1

TASK [OpenLDAP : Create organizational unit for groups] **************************************************************
changed: [ubuntu1]

TASK [OpenLDAP : Create groups] **************************************************************************************
changed: [ubuntu1] => (item=group1)
changed: [ubuntu1] => (item=group2)

PLAY RECAP ***********************************************************************************************************
ubuntu1                    : ok=15   changed=10   unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```
* При повторном запуске:
```BASH
local@sher-pc:~/Documents/ansible$ ansible-playbook playbook.yml -i hosts.txt 

PLAY [Install OpenLDAP] **********************************************************************************************

TASK [Gathering Facts] ***********************************************************************************************
ok: [ubuntu1]

TASK [OpenLDAP : include_tasks] **************************************************************************************
included: /home/local/Documents/ansible/OpenLDAP/tasks/installing_ldap.yml for ubuntu1

TASK [OpenLDAP : Update apt cache] ***********************************************************************************
changed: [ubuntu1]

TASK [OpenLDAP : Install OpenLDAP] ***********************************************************************************
ok: [ubuntu1]

TASK [OpenLDAP : Ensure slapd is running and enabled] ****************************************************************
ok: [ubuntu1]

TASK [OpenLDAP : include_tasks] **************************************************************************************
included: /home/local/Documents/ansible/OpenLDAP/tasks/configure_ldap.yml for ubuntu1

TASK [OpenLDAP : Replace config] *************************************************************************************
ok: [ubuntu1]

TASK [OpenLDAP : Restart slapd] **************************************************************************************
changed: [ubuntu1]

TASK [OpenLDAP : Create root base] ***********************************************************************************
ok: [ubuntu1]

TASK [OpenLDAP : include_tasks] **************************************************************************************
included: /home/local/Documents/ansible/OpenLDAP/tasks/adding_users.yml for ubuntu1

TASK [OpenLDAP : Create organizational unit for users] ***************************************************************
ok: [ubuntu1]

TASK [OpenLDAP : Create LDAP users] **********************************************************************************
ok: [ubuntu1] => (item=user1)
ok: [ubuntu1] => (item=user2)

TASK [OpenLDAP : include_tasks] **************************************************************************************
included: /home/local/Documents/ansible/OpenLDAP/tasks/adding_groups.yml for ubuntu1

TASK [OpenLDAP : Create organizational unit for groups] **************************************************************
ok: [ubuntu1]

TASK [OpenLDAP : Create groups] **************************************************************************************
ok: [ubuntu1] => (item=group1)
ok: [ubuntu1] => (item=group2)

PLAY RECAP ***********************************************************************************************************
ubuntu1                    : ok=15   changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

Тесты на ноде:
* После первого запуска:
```BASH
root@vm1:~# ldapsearch -Q -Y EXTERNAL -H ldapi:/// -b cn=config "(olcDatabase={1}mdb)" olcSuffix olcRootDN olcRootPW
# extended LDIF
#
# LDAPv3
# base <cn=config> with scope subtree
# filter: (olcDatabase={1}mdb)
# requesting: olcSuffix olcRootDN olcRootPW 
#

# {1}mdb, config
dn: olcDatabase={1}mdb,cn=config
olcSuffix: dc=example,dc=com
olcRootDN: cn=admin,dc=example,dc=com
olcRootPW: {SSHA}J3s1NeRgOzXocPinAkRRAzNHUmJHhr0z

# search result
search: 2
result: 0 Success

# numResponses: 2
# numEntries: 1

root@vm1:~# ldapwhoami -x -H ldap:/// -D "cn=admin,dc=example,dc=com" -w "changeme"
dn:cn=admin,dc=example,dc=com

root@vm1:~# ldapwhoami -x -H ldap:/// -D "cn=admin,dc=example,dc=com" -w "changeme"
dn:cn=admin,dc=example,dc=com
root@vm1:~# ldapsearch -x -b "dc=example,dc=com" -s base "(objectClass=organization)"
# extended LDIF
#
# LDAPv3
# base <dc=example,dc=com> with scope baseObject
# filter: (objectClass=organization)
# requesting: ALL
#

# example.com
dn: dc=example,dc=com
o: Example Organization
dc: example
objectClass: organization
objectClass: dcObject
objectClass: top

# search result
search: 2
result: 0 Success

# numResponses: 2
# numEntries: 1

root@vm1:~# ldapsearch -x -b "ou=users,dc=example,dc=com" "(objectClass=inetOrgPerson)"
# extended LDIF
#
# LDAPv3
# base <ou=users,dc=example,dc=com> with scope subtree
# filter: (objectClass=inetOrgPerson)
# requesting: ALL
#

# user1, users, example.com
dn: uid=user1,ou=users,dc=example,dc=com
uid: user1
cn: UserN1 UserS1
givenName: UserN1
sn: UserS1
mail: user1@example.com
uidNumber: 10001
gidNumber: 10001
homeDirectory: /home/user1
loginShell: /bin/bash
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: top

# user2, users, example.com
dn: uid=user2,ou=users,dc=example,dc=com
uid: user2
cn: UserN2 UserS2
givenName: UserN2
sn: UserS2
mail: user2@example.com
uidNumber: 10002
gidNumber: 10002
homeDirectory: /home/user2
loginShell: /bin/bash
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: top

# search result
search: 2
result: 0 Success

# numResponses: 3
# numEntries: 2

root@vm1:~# ldapsearch -x -b "ou=groups,dc=example,dc=com" "(objectClass=groupOfNames)"
# extended LDIF
#
# LDAPv3
# base <ou=groups,dc=example,dc=com> with scope subtree
# filter: (objectClass=groupOfNames)
# requesting: ALL
#

# group1, groups, example.com
dn: cn=group1,ou=groups,dc=example,dc=com
cn: group1
member: uid=user1,ou=users,dc=example,dc=com
description: First group
objectClass: groupOfNames
objectClass: top

# group2, groups, example.com
dn: cn=group2,ou=groups,dc=example,dc=com
cn: group2
member: uid=user2,ou=users,dc=example,dc=com
description: Second group
objectClass: groupOfNames
objectClass: top

# search result
search: 2
result: 0 Success

# numResponses: 3
# numEntries: 2
```
* После повторного запуска:
```BASH
root@vm1:~# ldapsearch -Q -Y EXTERNAL -H ldapi:/// -b cn=config "(olcDatabase={1}mdb)" olcSuffix olcRootDN olcRootPW
# extended LDIF
#
# LDAPv3
# base <cn=config> with scope subtree
# filter: (olcDatabase={1}mdb)
# requesting: olcSuffix olcRootDN olcRootPW 
#

# {1}mdb, config
dn: olcDatabase={1}mdb,cn=config
olcSuffix: dc=example,dc=com
olcRootDN: cn=admin,dc=example,dc=com
olcRootPW: {SSHA}J3s1NeRgOzXocPinAkRRAzNHUmJHhr0z

# search result
search: 2
result: 0 Success

# numResponses: 2
# numEntries: 1

root@vm1:~# ldapwhoami -x -H ldap:/// -D "cn=admin,dc=example,dc=com" -w "changeme"
dn:cn=admin,dc=example,dc=com

root@vm1:~# ldapwhoami -x -H ldap:/// -D "cn=admin,dc=example,dc=com" -w "changeme"
dn:cn=admin,dc=example,dc=com
root@vm1:~# ldapsearch -x -b "dc=example,dc=com" -s base "(objectClass=organization)"
# extended LDIF
#
# LDAPv3
# base <dc=example,dc=com> with scope baseObject
# filter: (objectClass=organization)
# requesting: ALL
#

# example.com
dn: dc=example,dc=com
o: Example Organization
dc: example
objectClass: organization
objectClass: dcObject
objectClass: top

# search result
search: 2
result: 0 Success

# numResponses: 2
# numEntries: 1

root@vm1:~# ldapsearch -x -b "ou=users,dc=example,dc=com" "(objectClass=inetOrgPerson)"
# extended LDIF
#
# LDAPv3
# base <ou=users,dc=example,dc=com> with scope subtree
# filter: (objectClass=inetOrgPerson)
# requesting: ALL
#

# user1, users, example.com
dn: uid=user1,ou=users,dc=example,dc=com
uid: user1
cn: UserN1 UserS1
givenName: UserN1
sn: UserS1
mail: user1@example.com
uidNumber: 10001
gidNumber: 10001
homeDirectory: /home/user1
loginShell: /bin/bash
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: top

# user2, users, example.com
dn: uid=user2,ou=users,dc=example,dc=com
uid: user2
cn: UserN2 UserS2
givenName: UserN2
sn: UserS2
mail: user2@example.com
uidNumber: 10002
gidNumber: 10002
homeDirectory: /home/user2
loginShell: /bin/bash
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: top

# search result
search: 2
result: 0 Success

# numResponses: 3
# numEntries: 2

root@vm1:~# ldapsearch -x -b "ou=groups,dc=example,dc=com" "(objectClass=groupOfNames)"
# extended LDIF
#
# LDAPv3
# base <ou=groups,dc=example,dc=com> with scope subtree
# filter: (objectClass=groupOfNames)
# requesting: ALL
#

# group1, groups, example.com
dn: cn=group1,ou=groups,dc=example,dc=com
cn: group1
member: uid=user1,ou=users,dc=example,dc=com
description: First group
objectClass: groupOfNames
objectClass: top

# group2, groups, example.com
dn: cn=group2,ou=groups,dc=example,dc=com
cn: group2
member: uid=user2,ou=users,dc=example,dc=com
description: Second group
objectClass: groupOfNames
objectClass: top

# search result
search: 2
result: 0 Success

# numResponses: 3
# numEntries: 2
```
