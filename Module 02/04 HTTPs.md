Поддержка протокола HTTPS

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
