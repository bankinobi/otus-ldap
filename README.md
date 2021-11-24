# Otus LDAP

* Установить FreeIPA;
* Написать Ansible playbook для конфигурации клиента;
* Настроить аутентификацию по SSH-ключам;
* Firewall должен быть включен на сервере и на клиенте.

## FreeIPA Server

Для корректной работы FreeIPA hostname у машин должен быть в формате fqdn

```
- name: Set FQDN hostname
  hostname: 
    name: '{{ ipa_server_fqdn }}'
```

Так же добавим в /etc/hosts записи с fqdn именами сервера и клиента 

```
- name: Add FQDN entries to /etc/hosts
  lineinfile:
    path: /etc/hosts
    line: "{{ item }}"
  loop:
    - '{{ ipa_server_ip }} {{ ipa_server_fqdn }} {{ ipa_server_fqdn }}'
    - '{{ ipa_client_ip }} {{ ipa_client_fqdn }} {{ ipa_client_fqdn }}'
```

Запустим *firewalld*

```
- name: Start and enable firewalld
  service:
    name: firewalld
    state: restarted
    enabled: true
```

Установка FreeIPA server происходит в *unattended* режиме? с передачей всех необходимых параметров в cli

```
- name: Install ipa server
  command: 
    cmd: |
      ipa-server-install -U
      --realm {{ ipa_realm }}
      --domain {{ ipa_domain }}
      --hostname={{ ipa_server_fqdn }}
      --ip-address={{ ipa_server_ip }}
      --setup-dns
      --auto-forwarders
      --no-reverse
      --mkhomedir
      -a ipa123123
      -p ipa123123
    creates: /etc/ipa/default.conf
```

Далее создаются sudorule OTUS c разрешением на выполнение всех команд. К этому правилу добавляем группу admins

```
- name: Add ipa sudorule 
  shell: ipa sudorule-add --cmdcat=all OTUS

- name: Grant admins group with OTUS sudorule
  shell: ipa sudorule-add-user --groups=admins OTUS
```

Создаем в ipa нового пользователя otus, и добавляем его в группу admins

```
- name: Create new ipa user
  shell: echo "ipa123123" | ipa user-add otus --first=Otus --last=Otusov --email=ootusov@otus.ru --shell=/bin/bash --sshpubkey="{{ ipa_user_pub_key }}" --password

- name: Add otus user to admins group
  shell: ipa group-add-member admins --users=otus
```

При создании пользователя в ipa передаем его публичный ssh ключ, а на сервер и клиент копируем приватный. При копировании используем *become_user: otus*, инициализируя таким образом пользователя в системе, и создавая для него homedir

```
- name: Create otus user .ssh dir
  file: 
   path: /home/otus/.ssh
   owner: otus
   group: otus
   state: directory
   mode: "0700"
  become: true
  become_user: otus
```

Последним шагом разрешаем входящие подключения к портам FreeIPA в firewall

```
- name: Add firewalld ipa service
  shell: |
    firewall-cmd --add-service=freeipa-ldap --add-service=freeipa-ldaps &&
    firewall-cmd --add-service=freeipa-ldap --add-service=freeipa-ldaps --permanent
```

## FreeIPA Client

Клиент устанавливается также в *unattended* режиме

```
- name: Install ipa client
  command: 
    cmd: |
      ipa-client-install -U
      --realm {{ ipa_realm }}
      --domain {{ ipa_domain }} 
      --server={{ ipa_server_fqdn }} 
      --ip-address={{ ipa_server_ip }} 
      --hostname={{ ipa_client_fqdn }} 
      --mkhomedir
      --force-ntpd
      -p admin 
      -w ipa123123 
    creates: /etc/ipa/default.conf
```

Далее, как и на сервере, создается homedir пользователя otus, и копируется приватный ключ.

## Тестирование

<details>
<summary>IPA Server</summary>

```
[vagrant@server ~]$ echo "ipa123123" | kinit admin
Password for admin@IPA.TEST:
[vagrant@server ~]$ ipa user-find --all
---------------
2 users matched
---------------
  dn: uid=admin,cn=users,cn=accounts,dc=ipa,dc=test
  User login: admin
  Last name: Administrator
  Full name: Administrator
  Home directory: /home/admin
  GECOS: Administrator
  Login shell: /bin/bash
  Principal alias: admin@IPA.TEST
  User password expiration: 20220221121722Z
  UID: 940800000
  GID: 940800000
  Account disabled: False
  Preserved user: False
  Member of groups: admins, trust admins
  Member of Sudo rule: All
  ipauniqueid: 86a50d72-4c56-11ec-88d9-5254004d77d3
  krbextradata: AALS25xhcm9vdC9hZG1pbkBJUEEuVEVTVAA=
  krblastpwdchange: 20211123121722Z
  objectclass: top, person, posixaccount, krbprincipalaux, krbticketpolicyaux, inetuser, ipaobject, ipasshuser, ipaSshGroupOfPubKeys

  dn: uid=otus,cn=users,cn=accounts,dc=ipa,dc=test
  User login: otus
  First name: Otus
  Last name: Otusov
  Full name: Otus Otusov
  Display name: Otus Otusov
  Initials: OO
  Home directory: /home/otus
  GECOS: Otus Otusov
  Login shell: /bin/bash
  Principal name: otus@IPA.TEST
  Principal alias: otus@IPA.TEST
  User password expiration: 20220221123616Z
  Email address: ootusov@otus.ru
  UID: 940800001
  GID: 940800001
  SSH public key: ssh-rsa
                  AAAAB3NzaC1yc2EAAAADAQABAAABAQDHnjhQ+M7O80/uh29sVAyHTp/ldL1WikH/tdlpQylYmYtFXOOps4zbY0WWxqBajg96mEAPLOjcV5cG8lHNjNNS2+N7oZMMJKOa9vsDrscXW8DvMiRykJi38RQm63Kxhw87SsqewqSj3VqOFIOEsf6NAYQUoe/eBdGNuhZowBbagzANOO74c1VxPUGFevsYNjh/cZNhjMRoZPO+bwIIJWAzBRIhpAZKUrkrWmKtnN4nNHU0SXKpjifJPBf/ofwWFKJRTii7o7ZIaIPADcbbjg/s1maISmXXs5Suti0DdchRMfPP2yi1q/LdHqFnHkGzCkep8TqypM3yBTB7KObdLQtt
                  otus@client.ipa.test
  SSH public key fingerprint: SHA256:PWH/XkNBgPzXOAS9rGHEi8d12+/utWkyNKrxHuo22PQ otus@client.ipa.test (ssh-rsa)
  Account disabled: False
  Preserved user: False
  Member of groups: admins, ipausers
  Member of Sudo rule: All
  ipauniqueid: 5c40c8fe-4c57-11ec-8523-5254004d77d3
  krbextradata: AAJA4Jxha2FkbWluZEBJUEEuVEVTVAA=
  krblastpwdchange: 20211123123616Z
  krbloginfailedcount: 0
  krbticketflags: 128
  mepmanagedentry: cn=otus,cn=groups,cn=accounts,dc=ipa,dc=test
  objectclass: top, person, organizationalperson, inetorgperson, inetuser, posixaccount, krbprincipalaux, krbticketpolicyaux, ipaobject, ipasshuser, ipaSshGroupOfPubKeys, mepOriginEntry
----------------------------
Number of entries returned 2
----------------------------
[vagrant@server ~]$ ll /home/
total 0
drwx------. 4 otus    otus    111 Nov 23 17:15 otus
drwx------. 6 vagrant vagrant 137 Nov 24 07:32 vagrant
[vagrant@server ~]$ grep otus /etc/passwd
[vagrant@server ~]$ sudo -i -u otus
[otus@server ~]$ ssh otus@client.ipa.test
Last login: Tue Nov 23 14:20:32 2021 from 192.168.50.20
[otus@client ~]$
```
</details>

<details>
<summary>IPA Client</summary>

```
[vagrant@client ~]$ echo "ipa123123" | kinit admin
Password for admin@IPA.TEST:
[vagrant@client ~]$ ipa user-find --all
---------------
2 users matched
---------------
  dn: uid=admin,cn=users,cn=accounts,dc=ipa,dc=test
  User login: admin
  Last name: Administrator
  Full name: Administrator
  Home directory: /home/admin
  GECOS: Administrator
  Login shell: /bin/bash
  Principal alias: admin@IPA.TEST
  User password expiration: 20220221121722Z
  UID: 940800000
  GID: 940800000
  Account disabled: False
  Preserved user: False
  Member of groups: admins, trust admins
  Member of Sudo rule: All
  ipauniqueid: 86a50d72-4c56-11ec-88d9-5254004d77d3
  krbextradata: AALS25xhcm9vdC9hZG1pbkBJUEEuVEVTVAA=
  krblastpwdchange: 20211123121722Z
  objectclass: top, person, posixaccount, krbprincipalaux, krbticketpolicyaux, inetuser, ipaobject, ipasshuser, ipaSshGroupOfPubKeys

  dn: uid=otus,cn=users,cn=accounts,dc=ipa,dc=test
  User login: otus
  First name: Otus
  Last name: Otusov
  Full name: Otus Otusov
  Display name: Otus Otusov
  Initials: OO
  Home directory: /home/otus
  GECOS: Otus Otusov
  Login shell: /bin/bash
  Principal name: otus@IPA.TEST
  Principal alias: otus@IPA.TEST
  User password expiration: 20220221123616Z
  Email address: ootusov@otus.ru
  UID: 940800001
  GID: 940800001
  SSH public key: ssh-rsa
                  AAAAB3NzaC1yc2EAAAADAQABAAABAQDHnjhQ+M7O80/uh29sVAyHTp/ldL1WikH/tdlpQylYmYtFXOOps4zbY0WWxqBajg96mEAPLOjcV5cG8lHNjNNS2+N7oZMMJKOa9vsDrscXW8DvMiRykJi38RQm63Kxhw87SsqewqSj3VqOFIOEsf6NAYQUoe/eBdGNuhZowBbagzANOO74c1VxPUGFevsYNjh/cZNhjMRoZPO+bwIIJWAzBRIhpAZKUrkrWmKtnN4nNHU0SXKpjifJPBf/ofwWFKJRTii7o7ZIaIPADcbbjg/s1maISmXXs5Suti0DdchRMfPP2yi1q/LdHqFnHkGzCkep8TqypM3yBTB7KObdLQtt
                  otus@client.ipa.test
  SSH public key fingerprint: SHA256:PWH/XkNBgPzXOAS9rGHEi8d12+/utWkyNKrxHuo22PQ otus@client.ipa.test (ssh-rsa)
  Account disabled: False
  Preserved user: False
  Member of groups: admins, ipausers
  Member of Sudo rule: All
  ipauniqueid: 5c40c8fe-4c57-11ec-8523-5254004d77d3
  krbextradata: AAJA4Jxha2FkbWluZEBJUEEuVEVTVAA=
  krblastpwdchange: 20211123123616Z
  krbloginfailedcount: 0
  krbticketflags: 128
  mepmanagedentry: cn=otus,cn=groups,cn=accounts,dc=ipa,dc=test
  objectclass: top, person, organizationalperson, inetorgperson, inetuser, posixaccount, krbprincipalaux, krbticketpolicyaux, ipaobject, ipasshuser, ipaSshGroupOfPubKeys, mepOriginEntry
----------------------------
Number of entries returned 2
----------------------------
[vagrant@client ~]$ sudo -i -u otus
[otus@client ~]$ ssh otus@server.ipa.test
Last login: Tue Nov 23 14:17:26 2021 from 192.168.50.21
```
</details>