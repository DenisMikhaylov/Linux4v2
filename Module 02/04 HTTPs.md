Поддержка протокола HTTPS
## Часть 1. Подготовка окружения Apache

На сервере Server
Настройка Apache

```
# a2enmod ssl
```
```
systemctl restart apache2
```
```
nano /etc/apache2/sites-available/default-ssl*
```
```
...
       SSLCertificateFile    /root/www.crt
       SSLCertificateKeyFile /root/www.key
...
```

```
a2ensite default-ssl
```
```
systemctl restart apache2
```



## Часть 2. Подготовка окружения Nginx


### Шаг 1.1. Установка Nginx на Gate
```bash
sudo apt update
sudo apt install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx
```

Проверка:
```bash
systemctl status nginx
curl -I http://localhost
```

### Шаг 1.2. Создание директорий для сертификатов
```bash
sudo mkdir -p /etc/nginx/ssl
sudo mkdir -p /var/www/html
```

### Шаг 1.3. Размещение сертификатов


```bash
# Копируем сертификат
sudo cp /путь/к/вашему/сертификату.crt /etc/nginx/ssl/gate.crt
# Копируем приватный ключ
sudo cp /путь/к/вашему/приватному.key /etc/nginx/ssl/gate.key
```


### Шаг 1.4. Установка прав доступа
```bash
sudo chmod 600 /etc/nginx/ssl/gate.key
sudo chmod 644 /etc/nginx/ssl/gate.crt
```

---

## Часть 3. Базовая конфигурация Nginx с SSL 

### Шаг 2.1. Создание тестовой веб-страницы
```bash
echo "<h1>HTTPS работает на gate!</h1>" | sudo tee /var/www/html/index.html
```

### Шаг 2.2. Создание конфигурационного файла сайта
```bash
sudo nano /etc/nginx/sites-available/gate-ssl
```

Содержимое:
```nginx
# Редирект с HTTP на HTTPS
server {
    listen 80;
    server_name gate;
    return 301 https://$server_name$request_uri;
}

# HTTPS сервер
server {
    listen 443 ssl;
    server_name gate;

    # Пути к сертификатам
    ssl_certificate     /etc/nginx/ssl/gate.crt;
    ssl_certificate_key /etc/nginx/ssl/gate.key;

    # Корневая директория
    root /var/www/html;
    index index.html;

    # Логи
    access_log /var/log/nginx/gate-access.log;
    error_log  /var/log/nginx/gate-error.log;
}
```

### Шаг 2.3. Активация сайта
```bash
sudo ln -s /etc/nginx/sites-available/gate-ssl /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default  # Отключаем дефолтный сайт
```

### Шаг 2.4. Проверка и перезагрузка
```bash
sudo nginx -t
sudo systemctl reload nginx
```

### Шаг 2.5. Тестирование
```bash
# Локальная проверка
curl -k https://localhost
curl -I https://gate

# Просмотр открытых портов
sudo netstat -tlnp | grep nginx
```

**Ожидаемый результат:** Порт 80 и 443 в статусе LISTEN.

---

## Часть 4. Продвинутые настройки безопасности

### Шаг 3.1. Усиление SSL-параметров
Отредактируйте файл конфигурации:
```bash
sudo nano /etc/nginx/sites-available/gate-ssl
```

Добавьте в блок `server` (порт 443) следующие параметры:

```nginx
    # Безопасность протоколов
    ssl_protocols TLSv1.2 TLSv1.3;
    
    # Набор шифров (с поддержкой Perfect Forward Secrecy)
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384';
    ssl_prefer_server_ciphers on;
    
    # Кэширование сессий (производительность)
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;
    
    # Заголовки безопасности
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
```

### Шаг 3.2. Полный пример конфигурации
```nginx
# HTTP редирект
server {
    listen 80;
    server_name gate;
    return 301 https://$server_name$request_uri;
}

# HTTPS сервер
server {
    listen 443 ssl;
    server_name gate;

    # SSL сертификаты
    ssl_certificate     /etc/nginx/ssl/gate.crt;
    ssl_certificate_key /etc/nginx/ssl/gate.key;

    # SSL настройки
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384';
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;

    # Заголовки безопасности
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;

    # Веб-контент
    root /var/www/html;
    index index.html;

    # Логи
    access_log /var/log/nginx/gate-access.log;
    error_log  /var/log/nginx/gate-error.log;
}
```

### Шаг 3.3. Применение настроек
```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## Часть 5. Диагностика и отладка 

### Шаг 4.1. Проверка SSL-соединения с помощью OpenSSL
```bash
# Просмотр сертификата, отдаваемого сервером
openssl s_client -connect gate:443 -showcerts

# Проверка поддержки конкретного протокола
openssl s_client -connect gate:443 -tls1_2
openssl s_client -connect gate:443 -tls1_3
```

### Шаг 4.2. Тестирование с помощью curl
```bash
# Без проверки сертификата
curl -k https://gate

# С подробным выводом
curl -vk https://gate

# Проверка поддержки HTTP/2 (если включено)
curl --http2 -k https://gate -I
```

### Шаг 4.3. Анализ логов
```bash
# Просмотр access логов
sudo tail -f /var/log/nginx/gate-access.log

# Просмотр error логов
sudo tail -f /var/log/nginx/gate-error.log

# Поиск ошибок SSL
sudo grep -i ssl /var/log/nginx/gate-error.log
```

### Шаг 4.4. Проверка прав доступа к ключам
```bash
ls -la /etc/nginx/ssl/
# Ключ должен быть 600, сертификат 644
```

