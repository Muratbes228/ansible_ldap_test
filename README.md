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

TASK [OpenLDAP : Update apt cache] *****************************************************************************************************************************
changed: [ubuntu1]
```
2) Произойдет проверка на наличие пакета splad на устройстве. Если пакет уже установлен на ноде, то он удаляется из системы вместе с предыдущими конфигами:
```BASH
TASK [OpenLDAP : Deleting old Openldap] ************************************************************************************************************************
changed: [ubuntu1]
```
3) Установится пакет slapd:
```BASH
TASK [OpenLDAP : Install OpenLDAP] *****************************************************************************************************************************
changed: [ubuntu1]
```
4) Сервис splad поднимется и добавится в автозапуск (systemctl enable):
```BASH
TASK [OpenLDAP : Ensure slapd is running and enabled] **********************************************************************************************************
ok: [ubuntu1]
```
5) Готовый конфиг OpenLDAP отправится на ноду/ноды, заменит текущий:
```BASH
included: /home/local/Documents/ansible/OpenLDAP/tasks/configure_ldap.yml for ubuntu1

TASK [OpenLDAP : Replacing config] *****************************************************************************************************************************
changed: [ubuntu1]
```
6) Сервис перезагрузится:
```BASH
TASK [OpenLDAP : Restarting slapd] *****************************************************************************************************************************
changed: [ubuntu1]
```
7) Загрузится конфиг корневой базы на ноду и применится:
```BASH
TASK [OpenLDAP : Loading root base file] ***********************************************************************************************************************
ok: [ubuntu1]

TASK [OpenLDAP : Adding root base file] ************************************************************************************************************************
changed: [ubuntu1]
```
8) Далее по такой же схеме будут загружены и применены файлы с пользователями и группами:
```BASH
included: /home/local/Documents/ansible/OpenLDAP/tasks/adding_users.yml for ubuntu1

TASK [OpenLDAP : Loading users file] ***************************************************************************************************************************
ok: [ubuntu1]

TASK [OpenLDAP : Adding users] *********************************************************************************************************************************
changed: [ubuntu1]

TASK [OpenLDAP : include_tasks] ********************************************************************************************************************************
included: /home/local/Documents/ansible/OpenLDAP/tasks/adding_groups.yml for ubuntu1

TASK [OpenLDAP : Loading groups file] **************************************************************************************************************************
ok: [ubuntu1]

TASK [OpenLDAP : Adding groups] ********************************************************************************************************************************
changed: [ubuntu1]
```

Основные правки в конфиге:
* olcSuffix: dc=example,dc=com (изначально стоит nodomain)
* olcRootDN: cn=admin,dc=example,dc=com
* olcRootPW: {SSHA}J3s1NeRgOzXocPinAkRRAzNHUmJHhr0z (тут для смены пароля использовал утилиту slappasswd и перевод в base64)

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
root@vm1:/etc/ldap# ldapwhoami -x -H ldap:/// -D "cn=admin,dc=example,dc=com" -w "changeme"
```
ожидаемый результат:
```BASH
dn:cn=admin,dc=example,dc=com
```
3) Наличие организации:
```BASH
root@vm1:/etc/ldap# ldapsearch -x -b "dc=example,dc=com" -s base "(objectClass=organization)"
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
root@vm1:/etc/ldap# ldapsearch -x -b "ou=People,dc=example,dc=com" "(objectClass=inetOrgPerson)"
```
ожидаемый результат:
```BASH
dn:cn=admin,dc=example,dc=com
root@vm1:/etc/ldap# ldapsearch -x -b "ou=People,dc=example,dc=com" "(objectClass=inetOrgPerson)"
# extended LDIF
#
# LDAPv3
# base <ou=People,dc=example,dc=com> with scope subtree
# filter: (objectClass=inetOrgPerson)
# requesting: ALL
#

# user1, People, example.com
dn: uid=user1,ou=People,dc=example,dc=com
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: inetOrgPerson
uid: user1
cn: User One
sn: One
givenName: User1
mail: user1@example.com

# user2, People, example.com
dn: uid=user2,ou=People,dc=example,dc=com
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: inetOrgPerson
uid: user2
cn: User Two
sn: Two
givenName: User2
mail: user2@example.com

# search result
search: 2
result: 0 Success

# numResponses: 3
# numEntries: 2
```
5) Добавление групп:
```BASH
root@vm1:/etc/ldap# ldapsearch -x -b "ou=Groups,dc=example,dc=com" "(objectClass=groupOfNames)"
```
ожидаемый результат:
```BASH
# extended LDIF
#
# LDAPv3
# base <ou=Groups,dc=example,dc=com> with scope subtree
# filter: (objectClass=groupOfNames)
# requesting: ALL
#

# group1, Groups, example.com
dn: cn=group1,ou=Groups,dc=example,dc=com
objectClass: top
objectClass: groupOfNames
cn: group1
description: First group
member: uid=user1,ou=People,dc=example,dc=com

# group2, Groups, example.com
dn: cn=group2,ou=Groups,dc=example,dc=com
objectClass: top
objectClass: groupOfNames
cn: group2
description: Second group
member: uid=user2,ou=People,dc=example,dc=com

# search result
search: 2
result: 0 Success

# numResponses: 3
# numEntries: 2
```

Почему было реализовано именно такое решение? Ранее я не сталкивался с установкой и использованием OpenLDAP, поэтому пришлось потратить время на то, чтобы изучить данный вопрос. Основной проблемой в процессе установки оказался интерактивный установщик. Через debconf получалось добавить все, кроме пароля. Далее была попытка добавить данные через ldapmodify, однако в данном случае конфиг редактировался не полностью, из-за чего сервис падал, жалуюся на проблемы в конфиге. Была идея собрать пакет из исходников, так как это, предположительно, упростило бы работу с его настройкой. Однако, в виду ограничений по времени, был выбран наиболее простой вариант - закинуть готовый конфиг. Данный вариант не идеален, однако его всегда можно переделать. 

В идеале, я бы сделал следующее:
1) Создал переменные (в идеале и их хранилище) для того, чтобы хранить основные данные.
2) Менял бы не весь конфиг, а только те строчки, которые нас интересуют.
3) Больше времени на прочтение документации, чтобы можно было найти какие-либо варианты для оптимизации взаимодействия с данным пакетом.
