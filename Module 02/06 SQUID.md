Установка и настройка Http proxy

1. Подключиться к Gate
2. Установка 

```
apt install squid
```

3. Настройка

```
cat /etc/squid/conf.d/my.conf
```
```
wc -l /etc/squid/squid.conf
```
или
```
cat /etc/squid/squid.conf
```

4. Посмотреть то что реально настроено в конфигурационном файле

```
grep -Ev '^$|^#' /etc/squid/squid.conf
```

5. Настроим правило
```
/etc/squid/squid.conf
```

```
...
# INSERT YOUR OWN RULE(S) HERE TO ALLOW ACCESS FROM YOUR CLIENTS
...
После этого блока

#http_access allow localnet
acl our_networks src 192.168.100.0/24
acl rambler dstdomain .rambler.ru

http_access deny rambler
http_access allow our_networks
...

```

6. Проверка

```
squid -k check
```
7. Переконфигурация

```
squid -k reconfigure
```
8. Подключиться к клиенту Windows. Настроить прокси для браузера и проверить интеренет
Просмотр логов

```
ls /var/log/squid
```

9. Настройка прокси для apt
```
nano /etc/apt/apt.conf
```
```
Acquire::http::proxy "http://gate.corp.local:3128";
Acquire::https::proxy "http://gate.corp.local:3128";
```
```
apt update
```
