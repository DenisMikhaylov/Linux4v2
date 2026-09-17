# Лабораторная работа: «Интеграция Debian-клиента в домен Active Directory с помощью Winbind»

**Цель работы:** Научиться настраивать Samba/Winbind на сервере `gate` (контроллер домена) и подключать Debian-клиент `client` к домену `CORP.LOCAL` с использованием Winbind для аутентификации доменных пользователей.

**Стенд:** Две виртуальные машины Debian:

| Узел | Роль | IP-адрес | FQDN |
|------|------|----------|------|
| **gate** | Контроллер домена (Samba AD DC) + DNS BIND + CA + Winbind | <ip gate> | `gate.corp.local` |
| **client** | Клиент, вводимый в домен | <ip client> | `client.corp.local` |

**Домен:** `CORP.LOCAL`
**NetBIOS-имя домена:** `CORP`
**Realm Kerberos:** `CORP.LOCAL`

**Предварительные требования:**
- На узле `gate` уже развёрнуты:
  - KDC Kerberos 5 (согласно предыдущим лабораторным работам).
  - DNS BIND с зоной `corp.local` и SRV-записями Kerberos.
  - Центр сертификации (CA) согласно лабораторной работе [«Использование PKI»](https://github.com/DenisMikhaylov/Linux4v2/blob/main/Module%2002/03%20PKI.md).
- Время на обоих узлах синхронизировано (разница не более 5 минут).
- Доступ к root или sudo на обоих узлах.

---

## Теоретическая справка

### Что такое Winbind?

**Winbind** — это компонент Samba, который позволяет Linux-системам использовать учётные записи и группы из домена Active Directory (или Samba AD DC) для аутентификации и разрешения имён. Он встраивается в NSS (Name Service Switch) и PAM (Pluggable Authentication Modules), что позволяет доменным пользователям входить в Linux-систему так же, как локальным.

### Как это работает?

1. **Kerberos** обеспечивает аутентификацию с контроллером домена.
2. **Samba/Winbind** связывает Linux-учётные записи с доменными учётными записями.
3. **NSS** позволяет системе «видеть» доменных пользователей и группы.
4. **PAM** позволяет доменным пользователям проходить аутентификацию при входе.

### ID Mapping

Winbind преобразует Windows SID (Security Identifier) в POSIX UID/GID с помощью **idmap-бэкендов**. Основные бэкенды:

| Бэкенд | Описание |
|--------|----------|
| **tdb** | Локальная база данных для локальных учётных записей и домена `*` |
| **rid** | Вычисляет UID из RID (последняя часть SID) — простой и детерминированный |
| **autorid** | Автоматически назначает диапазоны для доменов |
| **ad** | Читает UID/GID из атрибутов AD (требует расширения схемы) |

---

## Часть 1: Настройка сервера gate (контроллер домена)

### Шаг 1.1: Проверка работы KDC, DNS и CA

Убедитесь, что все предварительные компоненты работают:

```bash
# Проверка KDC
sudo systemctl status krb5-kdc --no-pager
sudo kadmin.local -q "listprincs" | head -10

# Проверка DNS
dig gate.corp.local A +short
dig _kerberos._udp.CORP.LOCAL SRV +short

# Проверка CA
sudo systemctl status apache2 --no-pager 2>/dev/null || true
wget -q http://gate.corp.local/ca.crt -O /tmp/ca.crt && echo "CA доступен"
```

### Шаг 1.2: Установка Samba и Winbind на gate

```bash
sudo apt update
sudo apt install -y samba krb5-user winbind libpam-winbind libnss-winbind smbclient
```

> **Важно:** Если на `gate` уже установлен Samba AD DC, этот шаг можно пропустить. Winbind на контроллере домена не требуется для его работы, но может быть установлен для тестирования.

### Шаг 1.3: Создание тестового доменного пользователя

Если в домене ещё нет тестового пользователя, создайте его через `kadmin.local`:

```bash
sudo kadmin.local
```

```
addprinc testuser
quit
```

Задайте пароль для `testuser`.

### Шаг 1.4: Проверка аутентификации на gate

```bash
kinit testuser@CORP.LOCAL
klist
kdestroy
```

---

## Часть 2: Подготовка клиента (client)

### Шаг 2.1: Настройка имени хоста

```bash
sudo hostnamectl set-hostname client.corp.local
hostname -f
# Ожидаем: client.corp.local
```

### Шаг 2.2: Настройка /etc/hosts

```bash
sudo nano /etc/hosts
```

Добавьте:
```
<ip gate>    gate.corp.local    gate
<ip client>    client.corp.local  client
127.0.0.1       localhost
```

### Шаг 2.3: Настройка DNS

Укажите контроллер домена как DNS-сервер:

```bash
sudo nano /etc/resolv.conf
```

Содержимое:
```
nameserver <ip gate>
search corp.local
```

Проверьте разрешение имён:
```bash
dig gate.corp.local +short
# Ожидаем: <ip gate>

dig _kerberos._udp.CORP.LOCAL SRV +short
# Ожидаем: 0 0 88 gate.corp.local.
```

### Шаг 2.4: Синхронизация времени

```bash
sudo apt update
sudo apt install -y chrony
sudo systemctl enable --now chrony
chronyc tracking
```

Убедитесь, что время синхронизировано с контроллером домена.

### Шаг 2.5: Установка пакетов

```bash
sudo apt install -y samba krb5-user winbind libpam-winbind libnss-winbind smbclient
```

### Шаг 2.6: Настройка Kerberos

```bash
sudo nano /etc/krb5.conf
```

Приведите к виду:

```ini
[libdefaults]
    default_realm = CORP.LOCAL
    dns_lookup_realm = false
    dns_lookup_kdc = true
    rdns = false
    ticket_lifetime = 24h
    renew_lifetime = 7d
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

Проверьте:
```bash
kinit testuser@CORP.LOCAL
klist
kdestroy
```

---

## Часть 3: Настройка Winbind на клиенте

### Шаг 3.1: Редактирование /etc/samba/smb.conf

```bash
sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.backup
sudo nano /etc/samba/smb.conf
```

Приведите файл к следующему виду (можно заменить содержимое полностью):

```ini
[global]
    workgroup = CORP
    realm = CORP.LOCAL
    security = ADS

    # Kerberos
    kerberos method = secrets and keytab

    # Winbind
    winbind use default domain = yes
    winbind offline logon = yes
    winbind refresh tickets = yes
    winbind enum users = yes
    winbind enum groups = yes

    # ID Mapping
    idmap config * : backend = tdb
    idmap config * : range = 3000-7999
    idmap config CORP : backend = rid
    idmap config CORP : range = 10000-19999

    # Шаблоны для домашних каталогов и оболочки
    template homedir = /home/%D/%U
    template shell = /bin/bash
```

### Шаг 3.2: Настройка NSS

```bash
sudo nano /etc/nsswitch.conf
```

Найдите строки `passwd` и `group` и приведите их к виду:

```
passwd:         files systemd winbind
group:          files systemd winbind
```

> **Примечание:** В Debian 12 может потребоваться добавить `winbind` в конец строки, как показано выше.

### Шаг 3.3: Настройка PAM

```bash
sudo pam-auth-update --enable winbind
sudo pam-auth-update --enable mkhomedir
```

Проверьте, что в `/etc/pam.d/common-auth` появилась строка с `pam_winbind.so`:

```bash
grep winbind /etc/pam.d/common-auth
```

Ожидаемый вывод:
```
auth [success=1 default=ignore] pam_winbind.so krb5_auth krb5_ccache_type=FILE cached_login try_first_pass
```

> **Примечание:** Пакет `libpam-winbind` автоматически настраивает PAM на Debian, добавляя необходимые строки.

### Шаг 3.4: Ввод клиента в домен

```bash
sudo net ads join -U Administrator
```

Введите пароль администратора домена.

Ожидаемый вывод:
```
Using short domain name -- CORP
Joined 'CLIENT' to dns domain 'corp.local'
```

Проверьте, что компьютер появился в домене:
```bash
sudo net ads testjoin
# Ожидаем: Join is OK
```

### Шаг 3.5: Запуск Winbind

```bash
sudo systemctl enable --now winbind
sudo systemctl status winbind --no-pager
```

Перезапустите Samba:
```bash
sudo systemctl restart smbd nmbd
```

---

## Часть 4: Проверка работы Winbind

### Шаг 4.1: Проверка доверия к домену

```bash
sudo wbinfo -t
# Ожидаем: checking the trust secret for domain CORP via RPC calls succeeded
```

### Шаг 4.2: Проверка списка пользователей и групп

```bash
sudo wbinfo -u | head -10
sudo wbinfo -g | head -10
```

### Шаг 4.3: Проверка разрешения имён через NSS

```bash
getent passwd testuser
id testuser
getent group "Domain Users"
```

Ожидаемый вывод для `getent passwd testuser`:
```
testuser:*:10001:10000:testuser:/home/CORP/testuser:/bin/bash
```

### Шаг 4.4: Проверка аутентификации через PAM

```bash
# Проверка аутентификации
wbinfo -a CORP\\testuser%пароль
# Ожидаем: plaintext password authentication succeeded
```

### Шаг 4.5: Проверка входа доменного пользователя

```bash
su - CORP\\testuser
# Введите пароль
whoami
# Ожидаем: corp\testuser
exit
```

Или через SSH:
```bash
ssh CORP\\testuser@localhost
```

При первом входе домашний каталог будет создан автоматически (благодаря `pam_mkhomedir`).

---

## Часть 5: Диагностика и устранение проблем

### 5.1. Полезные команды

```bash
# Проверка доверия к домену
wbinfo -t

# Список пользователей домена
wbinfo -u

# Список групп домена
wbinfo -g

# Преобразование SID в имя
wbinfo -s S-1-5-21-...

# Преобразование имени в SID
wbinfo -n CORP\\testuser

# Проверка PAM-аутентификации
wbinfo -a CORP\\testuser%password

# Проверка NSS
getent passwd testuser
id testuser

# Проверка Kerberos
kinit testuser@CORP.LOCAL
klist
kdestroy
```

### 5.2. Логи

```bash
# Логи Winbind
sudo journalctl -u winbind -f

# Логи Samba
sudo tail -f /var/log/samba/log.winbindd

# Отладка Winbind
sudo systemctl stop winbind
sudo winbindd -i -d 3
```

### 5.3. Типичные ошибки

| Ошибка | Причина | Решение |
|--------|---------|---------|
| `winbindd: no domain specified` | Домен не настроен в `smb.conf` | Проверьте `workgroup` и `realm` |
| `Could not get SID` | Проблема с DNS или Kerberos | Проверьте `dig gate.corp.local` и `kinit` |
| `Clock skew too great` | Рассинхронизация времени | Настройте `chrony` |
| `Failed to join domain` | Неверные учётные данные или DNS | Проверьте `net ads join -U Administrator` |
| `getent passwd` не показывает доменных пользователей | NSS не настроен | Добавьте `winbind` в `/etc/nsswitch.conf` |
| Аутентификация через PAM не работает | PAM-модуль не активирован | `pam-auth-update --enable winbind` |
| `wbinfo -t` не работает | Winbind не запущен | `sudo systemctl restart winbind` |

---

## Часть 6: Дополнительные задания

1. **Настройте offline-аутентификацию:** Убедитесь, что `winbind offline logon = yes` работает, и проверьте вход при отключённой сети.
2. **Добавьте доменную группу в sudoers:** Разрешите группе `Domain Admins` выполнять команды через `sudo`.
3. **Настройте автоматическое создание домашних каталогов** с правильными правами.
4. **Проверьте работу `wbinfo -a`** с неверным паролем и убедитесь, что аутентификация отклоняется.
5. **Изучите логи Winbind** и найдите записи о подключении к контроллеру домена.

---

---

**Желаю успешной сдачи! Вопросы приветствуются.**
