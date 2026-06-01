

# Лабораторная работа: Централизованное управление политикой паролей через PAM

## Цель работы

Изучить механизмы PAM (Pluggable Authentication Modules) для управления политикой сложности паролей, научиться настраивать модуль `pam_pwquality` и применять единые требования к паролям на нескольких хостах.



### Шаг 1. Установка необходимых пакетов

На всех трёх машинах выполните:

```bash
sudo apt update
sudo apt install -y pam-tools libpam-pwquality openssh-server
```

## Часть 1. Локальная настройка PAM (выполняется на всех машинах)

### Создание тестового пользователя

На каждой ВМ создайте пользователя `student`:

```bash
sudo useradd -m -s /bin/bash student
sudo passwd student
```
*(задайте временный пароль, например: student123)*

### Просмотр текущей конфигурации PAM

Изучите содержимое файла, отвечающего за смену паролей:

```bash
cat /etc/pam.d/common-password
```

Обратите внимание на строку с модулем `pam_pwquality.so`.

### Настройка политики сложности паролей

Отредактируйте файл `/etc/pam.d/common-password`. Найдите строку, содержащую `pam_pwquality.so`, и измените её следующим образом:

```bash
password requisite pam_pwquality.so retry=3 minlen=8 dcredit=-1 ucredit=-1 ocredit=-1 lcredit=-1
```

А также дополнительный параметр:
```bash
password requisite pam_pwquality.so retry=3 minlen=8 difok=3 reject_username
```

**Пояснение параметров:**
- `retry=3` — количество попыток ввода пароля
- `minlen=8` — минимальная длина пароля
- `dcredit=-1` — обязательное наличие хотя бы одной цифры
- `ucredit=-1` — обязательное наличие хотя бы одной заглавной буквы
- `lcredit=-1` — обязательное наличие хотя бы одной строчной буквы
- `ocredit=-1` — обязательное наличие хотя бы одного спецсимвола
- `difok=3` — количество символов, отличающихся от старого пароля
- `reject_username` — запрет использования имени пользователя в пароле




## Часть 3. Централизованное применение политики

### Применение единой политики на server и client

**На server и client** убедитесь, что конфигурация PAM загружена корректно:

```bash
cat /etc/pam.d/common-password | grep pwquality
```

### Тестирование на server

**На server** проверьте работу политики:

1. Переключитесь на пользователя `student`:
   ```bash
   sudo su - student
   ```

2. Попробуйте сменить пароль:
   ```bash
   passwd
   ```

3. Протестируйте следующие варианты паролей:
   - `qwerty` (слишком простой)
   - `password123` (нет заглавных и спецсимволов)
   - `Student123` (нет цифр? проверьте)
   - `Student123!` (должен подойти)
   - `Student` (имя пользователя в пароле)

---

# ЗАДАНИЕ ДЛЯ ВЫПОЛНЕНИЯ

## Задача

Администратор инфраструктуры установил требование: все пользователи на всех хостах (`gate`, `server`, `client`) должны создавать пароли, 
удовлетворяющие следующим условиям:

1. Минимальная длина пароля — **12 символов**
2. Пароль должен содержать символы **минимум из 3 различных классов** (строчные, заглавные, цифры, спецсимволы)
3. **Количество повторяющихся символов подряд** — не более 3 (например, запретить `aaaa`)
4. **Максимальная длина последовательных символов** — не более 4 (например, запретить `12345` или `abcded`)
5. Пароль **не должен быть похож на предыдущий** — минимум 5 отличий
6. Имя пользователя **не может быть частью пароля** (ни прямым, ни перевернутым)

Требуется реализовать описанную политику с помощью модуля PAM на всех трёх машинах и подтвердить её работоспособность.


# ОТВЕТ НА ЗАДАНИЕ

## Реализованная конфигурация

Для выполнения требований задания необходимо отредактировать файл `/etc/pam.d/common-password` на каждой машине (или только на gate с последующей синхронизацией).

### Итоговая строка конфигурации

```bash
password requisite pam_pwquality.so retry=3 minlen=12 difok=5 reject_username maxrepeat=3 maxsequence=4 minclass=3
```

### Дополнительная конфигурация в /etc/security/pwquality.conf

Для более тонкой настройки можно также создать или отредактировать файл `/etc/security/pwquality.conf`:

```ini
# /etc/security/pwquality.conf
minlen = 12
difok = 5
reject_username = yes
maxrepeat = 3
maxsequence = 4
minclass = 3
dictcheck = 1
usercheck = 1
enforcing = 1
local_users_only = 0
```

### Полный листинг измененного файла common-password (Debian 12)

```bash
# /etc/pam.d/common-password - password-related modules common to all services

# Стандартная первая часть с проверкой качества пароля
password requisite pam_pwquality.so retry=3 minlen=12 difok=5 reject_username maxrepeat=3 maxsequence=4 minclass=3

# SHA-512 хеширование и обновление shadow
password [success=1 default=ignore] pam_unix.so obscure use_authtok try_first_pass sha512

# Блокировка повторного использования старых паролей
password requisite pam_deny.so

# Конец блока
password required pam_permit.so
```

### Проверка работоспособности

Для проверки настроек выполните следующие тесты:

**Тест 1: Слишком короткий пароль**
```bash
$ passwd
Новый пароль: Abc123!
BAD PASSWORD: The password is shorter than 12 characters
```

**Тест 2: Меньше 3 классов символов**
```bash
$ passwd  
Новый пароль: Abcdefghijk12
BAD PASSWORD: The password fails the dictionary check - it is based on a dictionary word и содержит только 2 класса из 4
```

**Тест 3: Повторяющиеся символы**
```bash
$ passwd
Новый пароль: Aaaabbbccc12!
BAD PASSWORD: The password contains too many identical characters in a row
```

**Тест 4: Слишком длинная последовательность**
```bash
$ passwd
Новый пароль: 12345678AbcD!
BAD PASSWORD: The password contains too long of a monotonic character sequence
```

**Тест 5: Имя пользователя в пароле**
```bash
# Для пользователя student
$ passwd
Новый пароль: StudentStrong123!
BAD PASSWORD: The password contains the user name in some form
```

**Тест 6: Корректный пароль (должен пройти)**
```bash
$ passwd
Новый пароль: tR!bL3-4dM!nX9
Повторите новый пароль: tR!bL3-4dM!nX9
passwd: все токены аутентификации успешно обновлены.
```

### Дополнительно: скрипт для автоматического тестирования

```bash
#!/bin/bash
# test_password_policy.sh - тестирование политики паролей

TEST_USER="student"
PASSWORDS=(
    "short:123"                    # короткий
    "OnlyLettersAndNumbers"        # нет спецсимволов
    "AAAAbbbb1234!"               # повторяющиеся
    "1234567890Ab"                # последовательность
    "Student123!!!"               # имя пользователя
    "GoodP@ssw0rd2024!"           # должен пройти
)

echo "=== Тестирование политики паролей ==="
for pwd_entry in "${PASSWORDS[@]}"; do
    pwd="${pwd_entry#*:}"
    echo -n "Проверка: $pwd ... "
    echo -e "$pwd\n$pwd" | sudo -u $TEST_USER passwd 2>&1 | grep -q "BAD PASSWORD"
    if [ $? -eq 0 ]; then
        echo "ОТКЛОНЕН"
    else
        echo "ПРИНЯТ"
    fi
done
```
