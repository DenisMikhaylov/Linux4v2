App Armor

Установка
```
# apt install apparmor-utils -y
```
```
# aa-status
```
проверить профиля
```
# find /etc/apparmor.d/
```
посомтреть любой профиль профиль 
```
# nano /etc/apparmor.d/bin.man
```

Создание профиля "вручную"

проверить библиотеки приложения
```
# ldd /bin/bash

# ldd /bin/cat

# ldd /usr/bin/file

# man file
```
Созданеи профилья в ручную
```
# nano /etc/apparmor.d/usr.local.sbin.webd
```
```
/usr/local/sbin/webd {

  network inet stream,

  /usr/local/sbin/webd r,
#  /usr/bin/bash ix,
  /usr/bin/cat ix,
  /usr/bin/file ix,
  /etc/magic r,
  /usr/share/file/magic.mgc r,
  /usr/lib/file/magic.mgc r,

  /var/www/** r,

  /usr/lib/x86_64-linux-gnu/libtinfo* mr,
  /usr/lib/x86_64-linux-gnu/libdl* mr,
  /usr/lib/x86_64-linux-gnu/libc* mr,
  /usr/lib/x86_64-linux-gnu/libz* mr,
  /usr/lib/x86_64-linux-gnu/libmagic* mr,

}
```


Включение/выключение профиля
```
# aa-complain /usr/local/sbin/webd

# aa-status

# tail -f /var/log/audit/audit.log | grep usr.local.sbin.webd

# aa-enforce /usr/local/sbin/webd

# tail -f /var/log/audit/audit.log | grep usr.local.sbin.webd

# aa-disable /usr/local/sbin/webd

```
Создание профиля автоматически при помощи генерации для useradd

```
# aa-genprof /usr/sbin/useradd
```
просмотр созданного профиля.
```
# service apparmor restart
```
