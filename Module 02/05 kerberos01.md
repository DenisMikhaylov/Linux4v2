# Лабораторная работа: «Развертывание Kerberos 5 с интеграцией DNS BIND»

**Цель работы:** Научиться развертывать центр распространения ключей Kerberos 5 (KDC) на Debian, интегрировать его с DNS-сервером BIND для автоматического обнаружения KDC, а также настраивать клиентскую часть и проверять аутентификацию.

**Стенд:** Две виртуальные машины Debian:

| Узел | Роль | IP-адрес | FQDN |
|------|------|----------|------|
| **gate** | KDC (Kerberos) + DNS BIND | <ip gate> | `gate.corp.local` |
| **server** | Клиент Kerberos | <ip server> | `server.corp.local` |

**Домен:** `CORP.LOCAL`
**Realm:** `CORP.LOCAL`

**Требования:**
- Время на обоих узлах должно быть синхронизировано (разница не более 5 минут).
- Доступ к root или sudo на обоих узлах.

---

## Теоретическая справка

### Что такое Kerberos?

**Kerberos** — это сетевой протокол аутентификации, использующий концепцию «доверенной третьей стороны». Вместо передачи пароля по сети, Kerberos выдает клиенту **билеты (tickets)**, которые подтверждают его личность. Центральный сервер — **KDC (Key Distribution Center)** — состоит из двух служб: **AS** (выдает TGT) и **TGS** (выдает сервисные билеты).

### Зачем нужен DNS для Kerberos?

Kerberos-клиенты могут автоматически находить KDC с помощью специальных **SRV-записей** в DNS. Это избавляет от необходимости вручную прописывать адрес KDC в конфигурации каждого клиента. Записи имеют вид:

- `_kerberos._udp.CORP.LOCAL` — для UDP-запросов (порт 88)
- `_kerberos._tcp.CORP.LOCAL` — для TCP-запросов (порт 88)
- `_kpasswd._udp.CORP.LOCAL` — для смены пароля (порт 464)



### Ключевые понятия

| Термин | Значение |
|--------|----------|
| **Realm** | Административная область Kerberos (CORP.LOCAL) |
| **Principal** | Уникальная идентичность (ivan@CORP.LOCAL) |
| **KDC** | Центр выдачи ключей (AS + TGS) |
| **TGT** | Мастер-билет для получения других билетов |
| **Keytab** | Файл с долговременными ключами для сервисов |
| **kinit** | Утилита получения TGT |
| **klist** | Утилита просмотра билетов |
| **kdestroy** | Утилита удаления билетов |

---

## Часть 1: Подготовка узлов (на обоих серверах)

### Шаг 1.1: Настройка имён хостов

**На сервере gate:**
```bash
sudo hostnamectl set-hostname gate.corp.local
```

**На сервере server:**
```bash
sudo hostnamectl set-hostname server.corp.local
```

### Шаг 1.2: Настройка /etc/hosts

На **обоих** серверах добавьте записи в `/etc/hosts`:

```bash
sudo nano /etc/hosts
```

Добавьте:
```
<ip gate>    gate.corp.local    gate
<ip server>    server.corp.local  server
```

Проверьте:
```bash
hostname -f
# На gate: gate.corp.local
# На server: server.corp.local
```

### Шаг 1.3: Синхронизация времени

На **обоих** серверах установите и запустите `chrony`:

```bash
sudo apt update
sudo apt install -y chrony
sudo systemctl enable --now chrony
```

Проверьте синхронизацию:
```bash
chronyc tracking
```

---

## Часть 2: Развертывание DNS BIND на сервере gate

### Шаг 2.1: Установка BIND

```bash
sudo apt install -y bind9 bind9utils bind9-doc
```

### Шаг 2.2: Настройка глобальных параметров BIND

Отредактируйте `/etc/bind/named.conf.options`:

```bash
sudo nano /etc/bind/named.conf.options
```

Приведите файл к следующему виду:

```
acl internals {
    127.0.0.0/8;
    192.168.1.0/24;
};

options {
    directory "/var/cache/bind";

    // Отключаем проверку DNSSEC для внутренней зоны
    dnssec-validation no;

    // Слушаем только на внутреннем интерфейсе
    listen-on port 53 { <ip gate>; 127.0.0.1; };
    listen-on-v6 { none; };

    // Разрешаем запросы только из внутренней сети
    allow-query { internals; };
    allow-query-cache { internals; };
    allow-recursion { internals; };

    // Форвардеры для внешних запросов
    forwarders {
        8.8.8.8;
        8.8.4.4;
    };

    // Минимальные ответы для совместимости
    minimal-responses yes;
};
```

### Шаг 2.3: Создание зоны прямого просмотра

Отредактируйте `/etc/bind/named.conf.local`:

```bash
sudo nano /etc/bind/named.conf.local
```

Добавьте:

```
zone "corp.local" {
    type master;
    file "/etc/bind/db.corp.local";
    allow-update { none; };
};
```

### Шаг 2.4: Создание файла зоны

Скопируйте шаблон:

```bash
sudo cp /etc/bind/db.local /etc/bind/db.corp.local
sudo nano /etc/bind/db.corp.local
```

Приведите файл к следующему виду:

```
$TTL    604800
@       IN      SOA     gate.corp.local. admin.corp.local. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      gate.corp.local.
@       IN      A       <ip gate>
gate    IN      A       <ip gate>
server  IN      A       <ip server>
```

### Шаг 2.5: Добавление SRV-записей Kerberos

Добавьте в конец файла `db.corp.local` следующие записи:

```
; Kerberos KDC discovery
_kerberos._udp.CORP.LOCAL.    IN  SRV  0 0 88  gate.corp.local.
_kerberos._tcp.CORP.LOCAL.    IN  SRV  0 0 88  gate.corp.local.
_kpasswd._udp.CORP.LOCAL.     IN  SRV  0 0 464 gate.corp.local.
_kpasswd._tcp.CORP.LOCAL.     IN  SRV  0 0 464 gate.corp.local.

; Kerberos master server
_kerberos-master._udp.CORP.LOCAL. IN SRV 0 0 88 gate.corp.local.
_kerberos-master._tcp.CORP.LOCAL. IN SRV 0 0 88 gate.corp.local.
```

> **Важно:** Имя realm в SRV-записях должно быть в **ВЕРХНЕМ РЕГИСТРЕ** (`CORP.LOCAL`), так как Kerberos чувствителен к регистру в именах realm. 

### Шаг 2.6: Создание зоны обратного просмотра (рекомендуется)

Для корректной работы Kerberos часто требуется обратное разрешение имён. Создайте зону для сети `192.168.1.0/24`.

В `/etc/bind/named.conf.local` добавьте:

```
zone "1.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192.168.1";
    allow-update { none; };
};
```

Создайте файл зоны:

```bash
sudo nano /etc/bind/db.192.168.1
```

Содержимое:

```
$TTL    604800
@       IN      SOA     gate.corp.local. admin.corp.local. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      gate.corp.local.
10      IN      PTR     gate.corp.local.
20      IN      PTR     server.corp.local.
```

### Шаг 2.7: Проверка синтаксиса и запуск BIND

```bash
# Проверка конфигурации
sudo named-checkconf

# Проверка зон
sudo named-checkzone corp.local /etc/bind/db.corp.local
sudo named-checkzone 1.168.192.in-addr.arpa /etc/bind/db.192.168.1

# Запуск BIND
sudo systemctl enable --now bind9
sudo systemctl status bind9 --no-pager
```

### Шаг 2.8: Настройка resolv.conf на обоих серверах

На **gate** и **server** укажите DNS-сервер:

```bash
sudo nano /etc/resolv.conf
```

Содержимое:
```
nameserver <ip gate>
search corp.local
```

> **Примечание:** В современных Debian `/etc/resolv.conf` может управляться `systemd-resolved` или `resolvconf`. Для стабильности можно закомментировать управление и задать файл вручную, либо использовать `resolvconf`.

### Шаг 2.9: Проверка DNS

```bash
# Проверка прямого разрешения
dig gate.corp.local A +short
# Ожидаем: <ip gate>

dig server.corp.local A +short
# Ожидаем: <ip server>

# Проверка SRV-записей
dig _kerberos._udp.CORP.LOCAL SRV +short
# Ожидаем: 0 0 88 gate.corp.local.

# Проверка обратного разрешения
dig -x <ip gate> +short
# Ожидаем: gate.corp.local.
```

---

## Часть 3: Развертывание KDC на сервере gate

### Шаг 3.1: Установка пакетов KDC

```bash
sudo apt install -y krb5-kdc krb5-admin-server krb5-user
```

Во время установки укажите:
- **Default Kerberos version 5 realm:** `CORP.LOCAL`
- **Kerberos servers for your realm:** `gate.corp.local`
- **Administrative server for your Kerberos realm:** `gate.corp.local`

### Шаг 3.2: Настройка /etc/krb5.conf

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

> **Ключевой момент:** Параметр `dns_lookup_kdc = true` позволяет клиентам находить KDC через SRV-записи DNS. 

### Шаг 3.3: Инициализация базы данных KDC

```bash
sudo krb5_newrealm
```

При запросе введите **мастер-пароль** (запомните его — он понадобится для восстановления).

Запустите и включите службы:

```bash
sudo systemctl enable --now krb5-kdc krb5-admin-server
sudo systemctl status krb5-kdc --no-pager
```

### Шаг 3.4: Создание принципалов

```bash
sudo kadmin.local
```

Внутри консоли:

```
# Административный принципал
addprinc admin/admin

# Пользовательский принципал
addprinc student1

# Принципал для сервиса (например, host/server.corp.local)
addprinc -randkey host/server.corp.local

# Выход
quit
```

Проверьте список принципалов:

```bash
sudo kadmin.local -q "listprincs"
```

---

## Часть 4: Настройка клиента на сервере server

### Шаг 4.1: Установка клиентских утилит

```bash
sudo apt install -y krb5-user
```

При установке укажите те же параметры:
- Realm: `CORP.LOCAL`
- KDC: `gate.corp.local`
- Admin server: `gate.corp.local`

### Шаг 4.2: Настройка /etc/krb5.conf

Скопируйте файл с gate или создайте вручную:

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

### Шаг 4.3: Проверка аутентификации

```bash
# Получение TGT для пользователя student1
kinit student1
# Введите пароль

# Проверка билетов
klist
```

Ожидаемый вывод:
```
Ticket cache: FILE:/tmp/krb5cc_1000
Default principal: student1@CORP.LOCAL

Valid starting     Expires            Service principal
23/07/26 10:00:00  23/07/26 20:00:00  krbtgt/CORP.LOCAL@CORP.LOCAL
```

```bash
# Удаление билетов
kdestroy

# Проверка, что билеты удалены
klist
# Должно быть: No credentials cache found
```

### Шаг 4.4: Проверка автоматического обнаружения KDC через DNS

Убедитесь, что клиент находит KDC через DNS, а не через конфигурацию:

```bash
# Временно отключите dns_lookup_kdc в /etc/krb5.conf
# Установите dns_lookup_kdc = false и убедитесь, что kinit всё равно работает
# (если в [realms] указан KDC)

# Затем включите dns_lookup_kdc = true и удалите секцию [realms]
# Проверьте, что kinit работает только через DNS
```

---

## Часть 5: Диагностика и устранение проблем

### 5.1. Проверка SRV-записей

```bash
# Проверка всех Kerberos SRV-записей
dig SRV _kerberos._udp.CORP.LOCAL +short
dig SRV _kerberos._tcp.CORP.LOCAL +short
dig SRV _kpasswd._udp.CORP.LOCAL +short

# Использование host
host -t SRV _kerberos._udp.CORP.LOCAL
```

### 5.2. Трассировка Kerberos

```bash
# Подробный вывод kinit
KRB5_TRACE=/dev/stderr kinit -V student1
```

Это покажет, какие DNS-запросы выполняются и какой KDC используется.

### 5.3. Проверка BIND

```bash
# Статус BIND
sudo systemctl status bind9

# Логи BIND
sudo journalctl -u bind9 -f

# Проверка зоны
sudo named-checkzone corp.local /etc/bind/db.corp.local
```

### 5.4. Типичные ошибки

| Ошибка | Причина | Решение |
|--------|---------|---------|
| `Cannot contact any KDC` | SRV-записи не найдены или BIND не работает | Проверьте `dig SRV _kerberos._udp.CORP.LOCAL` |
| `Server not found in Kerberos database` | Принципал не создан | `sudo kadmin.local -q "listprincs"` |
| `Clock skew too great` | Рассинхронизация времени | Настройте `chrony` на обоих узлах |
| `KDC reply did not match expectations` | Ошибка в `krb5.conf` | Проверьте регистр realm и `domain_realm` |
| `No address associated with hostname` | DNS не разрешает имена | Проверьте `/etc/resolv.conf` |

---


---

