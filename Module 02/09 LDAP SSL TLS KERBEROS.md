# Лабораторная работа №17: «Защита OpenLDAP с помощью SSL/TLS и Kerberos»

**Цель работы:** Научиться защищать службу каталогов OpenLDAP с помощью шифрования трафика (SSL/TLS) и аутентификации через Kerberos (SASL/GSSAPI). Освоить настройку сертификатов, keytab-файлов и проверку защищённых соединений.

**Стенд:** Три виртуальные машины Debian:

| Узел | Роль | IP-адрес | FQDN |
|------|------|----------|------|
| **gate** | KDC (Kerberos) — уже развёрнут | <ip gate> | `gate.corp.local` |
| **server** | OpenLDAP (slapd) | <ip server> | `server.corp.local` |
| **client** | Клиент LDAP | <ip client> | `client.corp.local` |

**Realm:** `CORP.LOCAL`

**Важно:** KDC на `gate` уже настроен и работает. Нам нужно настроить OpenLDAP на `server` и проверить работу с `client`.

---

## Теоретическая справка

### Зачем защищать OpenLDAP?

По умолчанию OpenLDAP передаёт данные в открытом виде по порту 389. Это означает, что логины, пароли и содержимое каталога могут быть перехвачены. Для защиты используются два уровня:

1. **SSL/TLS** — шифрует канал связи между клиентом и сервером.
2. **Kerberos/SASL/GSSAPI** — обеспечивает аутентификацию без передачи пароля по сети.

### StartTLS vs LDAPS

| Характеристика | StartTLS | LDAPS |
|----------------|----------|-------|
| Порт | 389 | 636 |
| Механизм | Апгрейд существующего соединения до TLS | TLS с самого начала |
| Статус | Уязвим к downgrade-атакам | Рекомендуется |

### Как работает GSSAPI-аутентификация

1. Пользователь выполняет `kinit` и получает TGT от KDC.
2. Клиент запрашивает у KDC сервисный билет для принципала `ldap/server.corp.local@CORP.LOCAL`.
3. Клиент передаёт билет серверу LDAP через SASL/GSSAPI.
4. Сервер проверяет билет с помощью keytab-файла.
5. Аутентификация успешна — пароль не передавался по сети.

---

## Часть 1: Настройка SSL/TLS на сервере OpenLDAP (server)

### Шаг 1.1: Установка OpenLDAP и утилит

```bash
sudo apt update
sudo apt install -y slapd ldap-utils gnutls-bin ssl-cert
```

При установке `slapd` укажите:
- **Administrator password:** задайте пароль администратора LDAP.
- **DNS domain name:** `corp.local`
- **Organization name:** `Corp`
- **Database backend:** `MDB`

Проверьте статус:
```bash
sudo systemctl status slapd --no-pager
```

### Шаг 1.2: Создание сертификатов

Создадим собственный удостоверяющий центр (CA) и сертификат сервера.

```bash
# Создание приватного ключа CA
sudo certtool --generate-privkey --outfile /etc/ssl/private/ca.key

# Шаблон для CA
cat > /tmp/ca.info << EOF
cn = Corp Local CA
ca
cert_signing_key
expiration_days = 3650
EOF

# Создание самоподписанного сертификата CA
sudo certtool --generate-self-signed \
    --load-privkey /etc/ssl/private/ca.key \
    --template /tmp/ca.info \
    --outfile /etc/ssl/certs/ca.pem

# Приватный ключ сервера LDAP
sudo certtool --generate-privkey --outfile /etc/ssl/private/ldap.key

# Шаблон для сертификата сервера
cat > /tmp/ldap.info << EOF
organization = Corp
cn = server.corp.local
tls_www_server
encryption_key
signing_key
expiration_days = 365
EOF

# Генерация сертификата сервера
sudo certtool --generate-certificate \
    --load-privkey /etc/ssl/private/ldap.key \
    --load-ca-certificate /etc/ssl/certs/ca.pem \
    --load-ca-privkey /etc/ssl/private/ca.key \
    --template /tmp/ldap.info \
    --outfile /etc/ssl/certs/ldap.pem
```

### Шаг 1.3: Установка прав на файлы

Файлы должны быть доступны пользователю `openldap`, под которым работает `slapd`.

```bash
# Группа ssl-cert имеет доступ к /etc/ssl/private
sudo adduser openldap ssl-cert

# Установка прав
sudo chown root:ssl-cert /etc/ssl/private/ldap.key
sudo chmod 640 /etc/ssl/private/ldap.key

sudo chown root:root /etc/ssl/certs/ldap.pem
sudo chmod 644 /etc/ssl/certs/ldap.pem

sudo chown root:root /etc/ssl/certs/ca.pem
sudo chmod 644 /etc/ssl/certs/ca.pem
```

### Шаг 1.4: Настройка TLS в slapd

Конфигурация OpenLDAP хранится в `cn=config`. Настроим TLS через `ldapmodify`.

```bash
cat > /tmp/tls.ldif << EOF
dn: cn=config
changetype: modify
add: olcTLSCACertificateFile
olcTLSCACertificateFile: /etc/ssl/certs/ca.pem
-
add: olcTLSCertificateFile
olcTLSCertificateFile: /etc/ssl/certs/ldap.pem
-
add: olcTLSCertificateKeyFile
olcTLSCertificateKeyFile: /etc/ssl/private/ldap.key
EOF

sudo ldapmodify -H ldapi:/// -Y EXTERNAL -f /tmp/tls.ldif
```

Ожидаемый вывод:
```
modifying entry "cn=config"
```

### Шаг 1.5: Включение LDAPS (порт 636)

Отредактируйте `/etc/default/slapd`:

```bash
sudo nano /etc/default/slapd
```

Найдите строку `SLAPD_SERVICES` и добавьте `ldaps:///`:

```
SLAPD_SERVICES="ldap:/// ldapi:/// ldaps:///"
```

Перезапустите службу:

```bash
sudo systemctl restart slapd
sudo systemctl status slapd --no-pager
```

### Шаг 1.6: Проверка TLS на сервере

```bash
# Проверка StartTLS
LDAPTLS_CACERT=/etc/ssl/certs/ca.pem ldapwhoami -H ldap://server.corp.local -ZZ -x
```

Ожидаемый вывод: `anonymous`

```bash
# Проверка LDAPS
LDAPTLS_CACERT=/etc/ssl/certs/ca.pem ldapwhoami -H ldaps://server.corp.local:636 -x
```

Ожидаемый вывод: `anonymous`

Если возникают ошибки, добавьте `-d 1` для отладки:
```bash
LDAPTLS_CACERT=/etc/ssl/certs/ca.pem ldapwhoami -H ldap://server.corp.local -ZZ -x -d 1 2>&1 | grep TLS
```

---

## Часть 2: Настройка Kerberos/GSSAPI на сервере OpenLDAP (server)

### Шаг 2.1: Установка клиента Kerberos

```bash
sudo apt install -y krb5-user libsasl2-modules-gssapi-mit sasl2-bin
```

При установке укажите:
- **Default realm:** `CORP.LOCAL`
- **KDC:** `gate.corp.local`
- **Admin server:** `gate.corp.local`

Скопируйте `/etc/krb5.conf` с KDC или создайте вручную:

```ini
[libdefaults]
    default_realm = CORP.LOCAL
    dns_lookup_realm = false
    dns_lookup_kdc = false
    rdns = false
    ticket_lifetime = 24h
    forwardable = true

[realms]
    CORP.LOCAL = {
        kdc = gate.corp.local
        admin_server = gate.corp.local
    }

[domain_realm]
    .corp.local = CORP.LOCAL
    corp.local = CORP.LOCAL
```

### Шаг 2.2: Создание принципала и keytab на KDC (gate)

На сервере **gate** выполните:

```bash
sudo kadmin.local
```

Внутри консоли:

```
addprinc -randkey ldap/server.corp.local@CORP.LOCAL
ktadd -k /tmp/ldap.keytab ldap/server.corp.local@CORP.LOCAL
quit
```

Скопируйте keytab на сервер **server**:

```bash
# На gate
sudo scp /tmp/ldap.keytab user@server.corp.local:/tmp/

# На server
sudo mv /tmp/ldap.keytab /etc/krb5.ldap.keytab
sudo chown root:openldap /etc/krb5.ldap.keytab
sudo chmod 640 /etc/krb5.ldap.keytab
```

> **Важно:** Права `640` и владелец `root:openldap` гарантируют, что только `slapd` сможет прочитать keytab.

### Шаг 2.3: Настройка SASL в slapd

Создайте файл конфигурации SASL:

```bash
sudo nano /etc/ldap/sasl2/slapd.conf
```

Содержимое:

```
keytab: /etc/krb5.ldap.keytab
mech_list: GSSAPI
```

### Шаг 2.4: Указание keytab в конфигурации slapd

Отредактируйте `/etc/default/slapd`:

```bash
sudo nano /etc/default/slapd
```

Раскомментируйте и измените строку:

```
export KRB5_KTNAME=/etc/krb5.ldap.keytab
```

Перезапустите службу:

```bash
sudo systemctl restart slapd
```

### Шаг 2.5: Проверка GSSAPI на сервере

```bash
# Получение TGT
kinit admin@CORP.LOCAL

# Проверка билетов
klist

# Аутентификация через GSSAPI
ldapwhoami -H ldap://server.corp.local -Y GSSAPI
```

Ожидаемый вывод:
```
dn:uid=admin,cn=corp.local,cn=gssapi,cn=auth
```

---

## Часть 3: Настройка клиента (client)

### Шаг 3.1: Установка клиентских утилит

```bash
sudo apt update
sudo apt install -y ldap-utils krb5-user libsasl2-modules-gssapi-mit
```

Скопируйте `/etc/krb5.conf` с KDC.

Скопируйте CA-сертификат с сервера **server**:

```bash
# На server
sudo scp /etc/ssl/certs/ca.pem user@client.corp.local:/tmp/

# На client
sudo mv /tmp/ca.pem /etc/ssl/certs/ca.pem
```

### Шаг 3.2: Настройка ldap.conf

```bash
sudo nano /etc/ldap/ldap.conf
```

Добавьте:

```
TLS_CACERT /etc/ssl/certs/ca.pem
TLS_REQCERT demand
SASL_MECH GSSAPI
```

### Шаг 3.3: Проверка TLS с клиента

```bash
# StartTLS
LDAPTLS_CACERT=/etc/ssl/certs/ca.pem ldapwhoami -H ldap://server.corp.local -ZZ -x
# Ожидаем: anonymous

# LDAPS
LDAPTLS_CACERT=/etc/ssl/certs/ca.pem ldapwhoami -H ldaps://server.corp.local:636 -x
# Ожидаем: anonymous
```

### Шаг 3.4: Проверка GSSAPI с клиента

```bash
# Получение TGT
kinit admin@CORP.LOCAL

# Проверка билетов
klist

# Аутентификация через GSSAPI
ldapwhoami -H ldap://server.corp.local -Y GSSAPI
# Ожидаем: dn:uid=admin,cn=corp.local,cn=gssapi,cn=auth
```

### Шаг 3.5: Комбинированная проверка (TLS + GSSAPI)

```bash
# StartTLS + GSSAPI
LDAPTLS_CACERT=/etc/ssl/certs/ca.pem ldapwhoami -H ldap://server.corp.local -ZZ -Y GSSAPI

# LDAPS + GSSAPI
LDAPTLS_CACERT=/etc/ssl/certs/ca.pem ldapwhoami -H ldaps://server.corp.local:636 -Y GSSAPI
```

Оба варианта должны вернуть DN аутентифицированного пользователя.

### Шаг 3.6: Поиск в каталоге через защищённое соединение

```bash
# Поиск с StartTLS + GSSAPI
LDAPTLS_CACERT=/etc/ssl/certs/ca.pem \
ldapsearch -H ldap://server.corp.local -ZZ -Y GSSAPI -b "dc=corp,dc=local" "(objectClass=*)"
```

---

## Часть 4: Диагностика и устранение проблем

### Проблемы TLS

| Ошибка | Причина | Решение |
|--------|---------|---------|
| `TLS init def ctx failed: -1` | Нет прав на файлы сертификатов | `sudo adduser openldap ssl-cert`; проверьте права `640` на ключ |
| `certificate verify failed` | Неверный CA | Укажите `TLS_CACERT /etc/ssl/certs/ca.pem` |
| `hostname mismatch` | CN не совпадает | Сертификат должен быть на `server.corp.local` |

### Проблемы Kerberos/GSSAPI

| Ошибка | Причина | Решение |
|--------|---------|---------|
| `SASL/GSSAPI authentication failed` | Нет keytab или неверный принципал | `klist -k /etc/krb5.ldap.keytab` |
| `Clock skew too great` | Рассинхронизация времени | Настройте NTP/chrony на всех узлах |
| `No worthy mechs found` | Не установлен GSSAPI-плагин | `apt install libsasl2-modules-gssapi-mit` |
| `Permission denied` (keytab) | Неверные права | `chown root:openldap`, `chmod 640` |

### Отладка

```bash
# Трассировка Kerberos
KRB5_TRACE=/dev/stderr ldapwhoami -H ldap://server.corp.local -Y GSSAPI

# Отладка TLS
LDAPTLS_CACERT=/etc/ssl/certs/ca.pem ldapwhoami -H ldap://server.corp.local -ZZ -x -d 1 2>&1 | grep TLS

# Просмотр keytab
sudo klist -k /etc/krb5.ldap.keytab

# Проверка поддерживаемых SASL-механизмов
ldapsearch -H ldap://server.corp.local -x -b "" -s base "(objectClass=*)" supportedSASLMechanisms
```

---
