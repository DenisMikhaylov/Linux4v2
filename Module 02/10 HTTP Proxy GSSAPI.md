
Задача 1 : настройка DNS


Настройки DNS клиентов на всех линукс
```
# nano /etc/resolv.conf
```
```
search corp.ru
nameserver 192.168.10.10
```
Задача 2 : Настройка Kerberos клиента
Подключится к Gate
Установка клиента Kerberos
```
# apt install krb5-user
```
Настройка на Kerbros Realm
```
# nano /etc/krb5.conf
```
```
[libdefaults]
    default_realm = CORP.RU
```
Тестирование
```
# kinit user1

# klist

# kdestroy
```
Задача 2: Установка, настройка и запуск пакета SQUID

Установка 

```
gate:~# apt install squid
```

Настройка

```
gate# nano /etc/squid/conf.d/my.conf
или
gate# nano /etc/squid/squid.conf
```

Посмотреть то чт реально настроено в конфигурационном файле

```
grep -Ev '^$|^#' /etc/squid/squid.conf
```

Настроим правило
```
/etc/squid/squid.conf
```

```
...
# INSERT YOUR OWN RULE(S) HERE TO ALLOW ACCESS FROM YOUR CLIENTS
...
#http_access allow localnet
acl our_networks src 192.168.X.0/24
http_access allow our_networks
...
```

Тестирование конфигурации и запуск
```
gate:~# squid -k check

gate:~# squid -k reconfigure

gate:~# tail -f /var/log/squid/access.log
```

Настройка DNS сервера

```
gate.corp.ru  A  192.168.10.1

1.10.168.192.in-addr.arpa PTR gate.corp.ru
```
Задача 3 : Аутентификация доступа к SQUID GSSAPI

На DC Добавляем пользователя в AD
```
Login: gatehttp
Password: Pa$$w0rd
Пароль не меняется и не устаревает
```
Создаем ключ сервиса http связывая его с фиктивным пользователем AD
Название сервиса HTTP обязательно заглавными буквами
```
C:\>setspn -L gatehttp

C:\>ktpass -princ HTTP/gate.corp.ru@CORP.ru -mapuser gatehttp -pass 'Pa$$w0rd' -out gatehttp.keytab

C:\>setspn -L gatehttp

C:\>setspn -Q HTTP/gate.corp.ru
```
Копируем ключ сервиса http сервер squid
```
WinSCP

C:\>pscp gatehttp.keytab root@gate:
```

Копируем ключи в системный keytab
```
root@gate:~# ktutil
ktutil: rkt /root/gatehttp.keytab
ktutil: list
ktutil: wkt /etc/krb5.keytab
ktutil: quit
```
```
root@gate:~# klist -ek /etc/krb5.keytab
```
Настраиваем права
```
gate# chmod +r /etc/krb5.keytab
```

Настройка сервиса SQUID на использование GSSAPI
```
gate# nano  /etc/squid/conf.d/my.conf
```
```
auth_param negotiate program /usr/lib/squid/negotiate_kerberos_auth -d

acl inetuser proxy_auth REQUIRED

http_access allow inetuser
```
