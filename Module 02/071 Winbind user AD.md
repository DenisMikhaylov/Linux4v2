Авторизация в режиме ADS/DOMAIN
```
gate# wbinfo -n user1     # может не работать на этом этапе
```
```
gate# nano /etc/samba/smb.conf
```
```
[global]
        ...
        winbind use default domain = Yes

        winbind expand groups = 1
        winbind enum users = yes
        winbind enum groups = yes
        winbind cache time = 36
        idmap config * : range = 20000-40000
        template homedir = /home/%U
        #use suitable shell (what abount /usr/sbin/nologin ?)
        template shell = /bin/sh
```
```
gate# service winbind restart
```

Авторизация на основе имени пользователя
```
gate# nano /etc/squid/conf.d/my.conf
```
```

auth_param negotiate program /usr/lib/squid/negotiate_kerberos_auth -d
#acl inetuser proxy_auth REQUIRED
acl inetuser proxy_auth user1@CORP.RU user2@CORP.RU
#acl inetuser proxy_auth_regex "/etc/squid/group1.acl"

http_access allow inetuser
```
Авторизация на основе членства в группе
```
gate# getent group group1 | cut -f4 -d: | tr "," "\n" | tee /etc/squid/group1.acl

```
```
gate# nano /etc/squid/conf.d/my.conf
```
```

auth_param negotiate program /usr/lib/squid/negotiate_kerberos_auth -d
#acl inetuser proxy_auth REQUIRED
#acl inetuser proxy_auth user1@CORP.RU user2@CORP.RU
acl inetuser proxy_auth_regex "/etc/squid/group1.acl"

http_access allow inetuser
```
```
gate# squid -k reconfigure
```
Для winbind авторизации
```
gate# ntlm_auth --username=user1 --require-membership-of=CORP\\group1
```

Использование WINBIND в библиотеке NSSWITCH

```
gate# wbinfo -S `wbinfo -n user1|cut -d' ' -f1`

gate# wbinfo -i user1
```
Информация вернеться но моежт быть с задержкой.
Добавим информацию для изминения последовательности

Установим библиотеки
```
apt install libnss-winbind
```
Настройка nnsswitch
```
gate# nano /etc/nsswitch.conf
```
```
...
passwd:         files systemd winbind
group:          files systemd winbind
shadow:         files winbind
...
```
Проверяем
```
gate# id user1
gate# getent passwd
gate# getent group
```
