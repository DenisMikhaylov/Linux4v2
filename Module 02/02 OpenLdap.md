# Лабораторная работа: Развёртывание и настройка OpenLDAP на Debian 12

## Цель работы

Освоить установку, базовую настройку и управление сервером OpenLDAP в Debian 12. Научиться создавать организационные единицы, пользователей и группы, а также проверять работу каталога с помощью утилит командной строки.

## Предварительные требования

- Виртуальная машина или физический сервер с чистым Debian 12 (Bookworm).
- Права root или пользователь с sudo.
- Настроенный FQDN (например, `ldap.corp.local`).
- Базовые навыки работы в командной строке Linux.

## Теоретическая справка

**OpenLDAP** — это свободная реализация протокола LDAP (Lightweight Directory Access Protocol), предназначенная для централизованного хранения информации о пользователях, группах и других ресурсах сети. В Debian 12 используется OpenLDAP версии 2.5+ с backend'ом MDB и динамической конфигурацией через `cn=config`.

**Ключевые компоненты:**
- `slapd` — основной демон сервера LDAP.
- `ldap-utils` — набор утилит командной строки (`ldapadd`, `ldapsearch`, `ldapmodify`).
- **LDIF** (LDAP Data Interchange Format) — текстовый формат для описания записей каталога.

---

## Часть 1. Подготовка системы

### 1.1. Настройка FQDN

Установите полное доменное имя для сервера:

```bash
sudo hostnamectl set-hostname ldap.corp.local
```

Отредактируйте файл `/etc/hosts`, добавив запись:

```bash
sudo nano /etc/hosts
```

Добавьте строку (замените IP на адрес вашей ВМ):

```
127.0.1.1   ldap.corp.local ldap
```

Проверьте:

```bash
hostname -f
ping -c 3 ldap.corp.local
```

Ожидаемый результат: `ldap.corp.local` и успешные ответы ping.

### 1.2. Обновление пакетов

```bash
sudo apt update && sudo apt upgrade -y
```

---

## Часть 2. Установка OpenLDAP

### 2.1. Установка пакетов

```bash
sudo apt install -y slapd ldap-utils
```

Во время установки будет предложено задать пароль администратора LDAP. **Запомните его** — он понадобится для управления каталогом.

### 2.2. Проверка статуса службы

```bash
sudo systemctl status slapd
```

Служба должна быть активна (`active (running)`). Если она не запущена:

```bash
sudo systemctl enable --now slapd
```

---

## Часть 3. Первичная конфигурация slapd

### 3.1. Реконфигурация через dpkg-reconfigure

По умолчанию при установке создаётся база с суффиксом, производным от FQDN. Для учебных целей удобнее задать понятный домен, например `corp.local`:

```bash
sudo dpkg-reconfigure slapd
```



### 3.2. Проверка базовой структуры

Проверьте, что сервер отвечает и база создана:

```bash
ldapsearch -x -H ldap://localhost -b "dc=corp,dc=local" -s base
```

В выводе должны быть атрибуты `namingContexts: dc=corp,dc=local` и `objectClass: domain`.

---

## Часть 4. Создание организационных единиц (OU)

Организационные единицы — это «папки» внутри каталога, которые упорядочивают записи. Создадим две OU: `people` для пользователей и `groups` для групп.

### 4.1. Создание LDIF-файла

```bash
nano ~/base.ldif
```

Содержимое:

```ldif
dn: ou=people,dc=corp,dc=local
objectClass: organizationalUnit
ou: people

dn: ou=groups,dc=corp,dc=local
objectClass: organizationalUnit
ou: groups
```

### 4.2. Загрузка OU в каталог

```bash
ldapadd -x -D "cn=admin,dc=corp,dc=local" -W -f ~/base.ldif
```

Система запросит пароль администратора. При успехе вы увидите:

```
adding new entry "ou=people,dc=corp,dc=local"
adding new entry "ou=groups,dc=corp,dc=local"
```

### 4.3. Проверка

```bash
ldapsearch -x -LLL -b "dc=corp,dc=local" "(objectClass=organizationalUnit)" dn
```

Должны отобразиться два DN: `ou=people,...` и `ou=groups,...`.

---

## Часть 5. Создание группы

Создадим группу `developers` с GID 5001.

### 5.1. Создание LDIF-файла

```bash
nano ~/addgroup.ldif
```

Содержимое:

```ldif
dn: cn=developers,ou=groups,dc=corp,dc=local
objectClass: posixGroup
cn: developers
gidNumber: 5001
```

### 5.2. Загрузка группы

```bash
ldapadd -x -D "cn=admin,dc=corp,dc=local" -W -f ~/addgroup.ldif
```

### 5.3. Проверка

```bash
ldapsearch -x -LLL -b "cn=developers,ou=groups,dc=corp,dc=local" cn gidNumber
```



---

## Часть 6. Создание пользователя

Создадим пользователя `ivan.petrov` с UID 10001.

### 6.1. Генерация хэша пароля

Пароль нельзя хранить в открытом виде. Сгенерируйте хэш:

```bash
slappasswd -h '{SSHA}'
```

Введите пароль дважды. Утилита выдаст строку вида `{SSHA}xxxxxxxxxxxxxxxxxxxxxxxx`. **Скопируйте её**.



### 6.2. Создание LDIF-файла

```bash
nano ~/adduser.ldif
```

Содержимое (замените `{SSHA}...` на полученный хэш):

```ldif
dn: uid=ivan.petrov,ou=people,dc=corp,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: ivan.petrov
cn: Ivan Petrov
sn: Petrov
givenName: Ivan
mail: ivan.petrov@corp.local
uidNumber: 10001
gidNumber: 5001
homeDirectory: /home/ivan.petrov
loginShell: /bin/bash
userPassword: {SSHA}xxxxxxxxxxxxxxxxxxxxxxxx
```

**Важно:** `gidNumber` пользователя должен совпадать с `gidNumber` группы, в которую он входит (в нашем случае — 5001).

### 6.3. Загрузка пользователя

```bash
ldapadd -x -D "cn=admin,dc=corp,dc=local" -W -f ~/adduser.ldif
```

### 6.4. Проверка

```bash
ldapsearch -x -LLL -b "ou=people,dc=corp,dc=local" "(uid=ivan.petrov)" cn mail uidNumber
```



---

## Часть 7. Проверка аутентификации

Убедимся, что пользователь может аутентифицироваться в LDAP:

```bash
ldapwhoami -x -D "uid=ivan.petrov,ou=people,dc=corp,dc=local" -W
```

Введите пароль пользователя. При успехе вы увидите:

```
dn:uid=ivan.petrov,ou=people,dc=corp,dc=local
```

---

## Часть 8. Просмотр всех объектов каталога

### 8.1. Все объекты

```bash
ldapsearch -x -LLL -D "cn=admin,dc=corp,dc=local" -W \
  -b "dc=corp,dc=local" \
  "(objectClass=*)"
```

### 8.2. Только DN

```bash
ldapsearch -x -LLL -D "cn=admin,dc=corp,dc=local" -W \
  -b "dc=corp,dc=local" \
  "(objectClass=*)" dn
```

### 8.3. Просмотр конфигурации

```bash
sudo ldapsearch -Y EXTERNAL -H ldapi:/// -LLL -b "cn=config" "(objectClass=*)" dn
```

### 8.4. Резервное копирование всей базы

```bash
sudo slapcat -b "dc=corp,dc=local" -l ~/ldap-backup.ldif
```



---

## Часть 9. Изменение и удаление записей (дополнительно)

### 9.1. Изменение атрибута

Создайте файл `modify.ldif`:

```ldif
dn: uid=ivan.petrov,ou=people,dc=corp,dc=local
changetype: modify
replace: mail
mail: i.petrov@corp.local
```

Примените:

```bash
ldapmodify -x -D "cn=admin,dc=corp,dc=local" -W -f ~/modify.ldif
```

### 9.2. Удаление пользователя

```bash
ldapdelete -x -D "cn=admin,dc=corp,dc=local" -W \
  "uid=ivan.petrov,ou=people,dc=corp,dc=local"
```

---

