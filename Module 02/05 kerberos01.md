

### Лабораторная работа №16: «Развертывание Kerberos 5 в домене CORP.LOCAL»

**Цель работы:** Научиться устанавливать и настраивать сервер Kerberos 5 (KDC) на Debian, настраивать клиентскую часть и проверять аутентификацию с помощью утилит `kinit`, `klist` и `kdestroy`.

**Теоретическая справка:**

**Основные понятия Kerberos:**
- **Realm (Сфера)** — административная граница, аналог домена. В нашей работе: `CORP.LOCAL`.
- **Principal (Принципал)** — уникальная идентичность в сфере. Формат: `имя@REALM` (пользователь) или `сервис/хост@REALM` (служба).
- **KDC (Key Distribution Center)** — центр выдачи ключей, состоящий из двух служб:
  - **AS (Authentication Server)** — выдает TGT (Ticket Granting Ticket) после аутентификации.
  - **TGS (Ticket Granting Service)** — выдает сервисные тикеты для доступа к службам.
- **TGT (Ticket Granting Ticket)** — «пропуск», который вы получаете после `kinit` и используете для получения тикетов к сервисам.
- **Keytab** — файл, содержащий долговременные ключи для служб, позволяющий им аутентифицироваться без пароля.

**Требования к окружению:**
- Два сервера Debian (можно использовать виртуальные машины).
- Настройки сети:
  - `gate` (KDC-сервер): IP-адрес `ip gate`
  - `server` (клиент): IP-адрес `ip server`
- Домены: `corp.local`
- **Важно:** Время на серверах должно быть синхронизировано (разница не более 5 минут), иначе Kerberos не будет работать.

---

### Шаг 1: Подготовка серверов (на обоих узлах)

Настройте имена хостов и добавьте записи в `/etc/hosts` на **обоих** серверах.

```bash
# На сервере gate
sudo hostnamectl set-hostname gate.corp.local

# На сервере server
sudo hostnamectl set-hostname server.corp.local

# На обоих серверах добавьте в /etc/hosts
sudo nano /etc/hosts
```

Добавьте следующие строки:
```
ip gate    gate.corp.local    gate
ip server    server.corp.local  server
```

Проверьте:
```bash
hostname -f
# Должно вывести: gate.corp.local (на gate) и server.corp.local (на server)
```

Установите `ntp` или `chrony` для синхронизации времени:
```bash
sudo apt update
sudo apt install -y chrony
sudo systemctl enable --now chrony
```

---

### Шаг 2: Установка Kerberos KDC на сервере gate

На **сервере `gate`** установите пакеты KDC и административного сервера:

```bash
sudo apt update
sudo apt install -y krb5-kdc krb5-admin-server krb5-user
```

Во время установки появятся диалоговые окна. Укажите:
- **Default Kerberos version 5 realm:** `CORP.LOCAL`
- **Kerberos servers for your realm:** `gate.corp.local`
- **Administrative server for your Kerberos realm:** `gate.corp.local`

---

### Шаг 3: Настройка конфигурации Kerberos на gate

Отредактируйте файл `/etc/krb5.conf`:

```bash
sudo nano /etc/krb5.conf
```

Приведите его к следующему виду:

```ini
[libdefaults]
    default_realm = CORP.LOCAL
    dns_lookup_realm = false
    dns_lookup_kdc = false
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



---

### Шаг 4: Инициализация базы данных KDC

На сервере `gate` инициализируйте базу данных Kerberos. Вам будет предложено задать **мастер-пароль** (запомните его):

```bash
sudo krb5_newrealm
```

При запросе введите мастер-пароль.

Запустите и включите службы:

```bash
sudo systemctl enable --now krb5-kdc krb5-admin-server
sudo systemctl status krb5-kdc --no-pager
```

---

### Шаг 5: Создание принципалов (пользователей и служб)

На сервере `gate` откройте административную консоль `kadmin.local` (доступна только локально с правами root):

```bash
sudo kadmin.local
```

Внутри консоли создайте:

1.  **Административного принципала** (для удаленного управления):
    ```
    addprinc admin/admin
    ```
    Введите пароль для администратора.

2.  **Пользовательского принципала** (например, для студента `student1`):
    ```
    addprinc student1
    ```
    Введите пароль для пользователя.

3.  **Принципала для службы SSH** на клиенте `server` (создадим заранее, чтобы клиент мог аутентифицироваться):
    ```
    addprinc -randkey host/server.corp.local
    ```
    Параметр `-randkey` означает, что ключ будет сгенерирован случайно (пароль не потребуется).

Выйдите из консоли:
```
quit
```

---

### Шаг 6: Настройка клиента на сервере server

На **сервере `server`** установите клиентские утилиты Kerberos:

```bash
sudo apt update
sudo apt install -y krb5-user
```

При установке укажите те же параметры, что и на KDC:
- Realm: `CORP.LOCAL`
- KDC: `gate.corp.local`
- Admin server: `gate.corp.local`

Скопируйте или создайте такой же файл `/etc/krb5.conf`, как на `gate`. **Важно:** файл должен быть идентичен на клиенте и сервере.

---

### Шаг 7: Проверка аутентификации

На **сервере `server`** выполните следующие команды:

1.  **Получите TGT (Ticket Granting Ticket)** для пользователя `student1`:
    ```bash
    kinit student1
    ```
    Введите пароль, который вы задали при создании принципала.

2.  **Проверьте наличие тикета**:
    ```bash
    klist
    ```
    Вы должны увидеть примерно следующее:
    ```
    Ticket cache: FILE:/tmp/krb5cc_1000
    Default principal: student1@CORP.LOCAL

    Valid starting     Expires            Service principal
    23/07/26 10:00:00  23/07/26 20:00:00  krbtgt/CORP.LOCAL@CORP.LOCAL
        renew until 30/07/26 10:00:00
    ```
    

3.  **Удалите тикет** (выход из системы):
    ```bash
    kdestroy
    ```

4.  **Убедитесь, что тикет удален**:
    ```bash
    klist
    ```
    Должно появиться сообщение об отсутствии тикетов.

---

### Шаг 8: Диагностика и устранение проблем

Если что-то не работает, используйте следующие команды для диагностики:

**Проверка статуса KDC:**
```bash
sudo systemctl status krb5-kdc --no-pager
```

**Просмотр логов KDC:**
```bash
sudo journalctl -u krb5-kdc -u krb5-admin-server -b --no-pager
```


**Трассировка запроса тикета:**
```bash
KRB5_TRACE=/dev/stderr kinit -V student1
```
Этот вывод покажет, какие имена хостов разрешаются и где происходит сбой.

**Список всех принципалов в базе:**
```bash
sudo kadmin.local -q "listprincs"
```


**Типичные ошибки:**
- **`Server not found in Kerberos database`** — несоответствие имени хоста в `krb5.conf` и при создании принципала.
- **`Clock skew too great`** — рассинхронизация времени. Проверьте работу `chrony` на обоих серверах.
- **`kinit: KDC reply did not match expectations`** — ошибка в `krb5.conf` (часто регистр в realm или domain_realm).

---

---

**Желаю успешной сдачи! Вопросы по настройке приветствуются.**
