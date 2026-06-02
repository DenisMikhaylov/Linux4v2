Добавить дополнительную виртуальную машину с debian

прописать имя server 2
прописать статический ip 192.168.10.30


Поднять BIND DNS
https://github.com/DenisMikhaylov/Linux-2/blob/main/Module%203%20/DNS.md

Добавить в конец
```
;_kerberos._udp      SRV     01 00 88 <Имя текущего сервера>
;_kerberos._tcp      SRV     01 00 88 <Имя текущего сервера>
;_kerberos           TXT     CORP.LOCAL
```
Установка

```
server2:~# apt install krb5-kdc krb5-admin-server
```

Настройка
```
nano /etc/krb5.conf
```
```
[libdefaults]
    default_realm = CORP.LOCAL
```

```
krb5_newrealm
```

```
# ls -l /var/lib/krb5kdc/
```

Перезапуск

```
systemctl restart krb5-kdc
```

Регистрация пользователей в  kerberos

```
# kadmin.local
kadmin.local:  addprinc user4
...
Enter password for principal "user4@CORP.LOCAL": Pa$$w0rd
Re-enter password for principal "user4@CORP.LOCAL": Pa$$w0rd
...
kadmin.local:  listprincs
...
kadmin.local: quit
```

Проверки
```
# kinit user4
# klist
```

