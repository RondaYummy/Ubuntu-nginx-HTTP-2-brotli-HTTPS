# Веб-сервер на Ubuntu з нуля: nginx, HTTP/2, brotli та HTTPS

Як налаштувати веб-сервер Nginx, зробити перенаправлення трафіку з 80 порту, на порт 3000 вашого додатку ( чи будь який
інший ), добавимо SSL сертифікат, та поясню як налаштувати DNS records, якщо ви хочете підключити домен. Опишу базові
моменти security aspect, такі як заборона логіну через root, лоігн по вашому публічному ключу та фаєрвол.

Перевірено на Ubuntu 20.04 LTS

## Настройка root

Підключаємось

```
ssh root@IP
```

Міняємо пароль

```
passwd
```

Обновляемо систему

```
sudo apt-get update
sudo apt-get upgrade
sudo apt-get dist-upgrade
sudo apt-get autoremove
```

Добавляемо користувача

```
adduser USER
```

Даємо права sudo

```
adduser USER sudo
```

Відкриваємо налаштування ssh

```
nano /etc/ssh/sshd_config
```

Забороняємо логін root

```
PermitRootLogin no
```

Рестартуємо ssh

```
systemctl restart sshd
```

Виходимо

```
exit
```

## Налаштування користувача

Підключаємося

```
ssh USER@IP
```

Робимо папку .ssh для ключа

```
mkdir -p ~/.ssh
```

Виводимо та копіюємо локальний ключ

```
cat ~/.ssh/id_rsa.pub
```

Вставляємо його на сервері

```
nano ~/.ssh/authorized_keys
```

Виставляємо права ( Щоб ніхто туди не добрався 🙂 )

```
sudo chmod -R 700 ~/.ssh/
```

Відкриваємо налаштування ssh

```
sudo nano /etc/ssh/sshd_config
```

Відключаємо вхід за паролем

```
PasswordAuthentication no
```

Рестартуємо ssh

```
sudo systemctl restart sshd
```

Відкриваємо налаштування sudo

```
sudo visudo
```

Налаштовуємо sudo без пароля ( щоб не просило весь час пароль, якщо виконуємо дії від root )

```
%sudo   ALL=(ALL:ALL) NOPASSWD:ALL
```

## Встановлення сервера

Встановлюємо файрвол

```
sudo apt install ufw
```

Додаємо репозиторій

```
sudo apt-add-repository -y ppa:hda-me/nginx-stable
```

Встановлюємо сервер та модулі

```
sudo apt-get install brotli nginx nginx-module-brotli
```

Ремонтуємо сервіс nginx 👀 ( Переважно проблема на Ubintu 18, на 20.04 все працює )

```
sudo systemctl unmask nginx.service
```

Відкриваємо налаштування сервера

```
sudo nano /etc/nginx/nginx.conf
```

Включаємо brotli

```
load_module modules/ngx_http_brotli_filter_module.so;
load_module modules/ngx_http_brotli_static_module.so;
```

Налаштовуємо brotli

```
    # Brotli
        brotli on;
        brotli_comp_level 6;

        brotli_types
          text/xml
          image/svg+xml
          application/x-font-ttf
          image/vnd.microsoft.icon
          application/x-font-opentype
          application/json
          font/eot
          application/vnd.ms-fontobject
          application/javascript
          font/otf
          application/xml
          application/xhtml+xml
          text/javascript
          application/x-javascript
          text/$;
```

## Налаштування файрвола

Перевіряємо статус

```
sudo ufw status
```

Перевіряємо список додатків

```
sudo ufw app list
```

```
Available applications:
  Nginx Full
  Nginx HTTP
  Nginx HTTPS
  OpenSSH
```

Додаємо nginx в ufw, якщо його немає

```
sudo nano /etc/ufw/applications.d/nginx.ini
```

```
[Nginx HTTP]
title=Web Server
description=Enable NGINX HTTP traffic
ports=80/tcp

[Nginx HTTPS] \
title=Web Server (HTTPS) \
description=Enable NGINX HTTPS traffic
ports=443/tcp

[Nginx Full]
title=Web Server (HTTP,HTTPS)
description=Enable NGINX HTTP and HTTPS traffic
ports=80,443/tcp
```

Перевіряємо список додатків

```
sudo ufw app list
```

Включаємо

```
sudo ufw enable
```

Дозволяємо nginx

```
sudo ufw allow 'Nginx Full'
```

Видалити надлишковий дозвіл профілю «Nginx HTTP»:

```
sudo ufw delete allow 'Nginx HTTP'
```

Дозволяється OpenSSH

```
sudo ufw allow 'OpenSSH'
```

Перевіряємо статус

```
sudo ufw status
```

```
Status: active

To                         Action      From
--                         ------      ----
Nginx Full                 ALLOW       Anywhere
OpenSSH                    ALLOW       Anywhere
Nginx Full (v6)            ALLOW       Anywhere (v6)
OpenSSH (v6)               ALLOW       Anywhere (v6)
```

## Установка certbot

Додаємо джерела

```
sudo apt-get install software-properties-common
sudo add-apt-repository ppa:certbot/certbot
```

Оновлюємося

```
sudo apt-get update
```

Ставимо certbot

sudo apt-get install python-certbot-nginx ( Old )

```
sudo apt install certbot python3-certbot-nginx
```

Генеруємо ключ

```
sudo openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048
```

Створюємо папку для сніпетів nginx ( для зручності )

```
sudo mkdir -p /etc/nginx/snippets/
```

Створюємо сніпет для SSL

```
sudo nano /etc/nginx/snippets/ssl-params.conf
```

```
ssl_session_timeout 1d;
ssl_session_cache shared:SSL:10m;
ssl_session_tickets off;

ssl_dhparam /etc/ssl/certs/dhparam.pem;

ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
ssl_prefer_server_ciphers off;

add_header Strict-Transport-Security "max-age=63072000" always;
```

## Додавання сайту

Створюємо конфіг сайту

```
sudo nano /etc/nginx/sites-available/example.com.conf
```

```
server {
    listen 80;
    listen [::]:80;

    server_name example.com www.example.com;
}
```

Створіть символьне посилання до файлу конфігурації з директорії /etc/nginx/sites-enabled/ до директорії
/etc/nginx/sites-available/, виконавши наступну команду

```
sudo ln -s /etc/nginx/sites-available/example.com.conf /etc/nginx/sites-enabled/
```

Перевіряємо конфіги nginx на помилки

```
sudo nginx -t
```

Рестартуємо nginx

```
sudo systemctl restart nginx
```

## Додавання сертифіката

Запускаємо certbot

```
sudo certbot --nginx certonly
```

Оновлюємо конфіг сайту

```
sudo nano /etc/nginx/sites-available/example.com.conf
```

Тут ми зробимо також редірект з 80 порту, на порт, на якому працює наш бекенд, у моєму випадку це 3000 та добавимо
сертифікати SSL

```
server {
    listen 80;
    listen [::]:80;

    server_name example.com www.example.com;
    return 301 https://example.com$request_uri;
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;

    server_name www.example.com;
    return 301 https://example.com$request_uri;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_trusted_certificate /etc/letsencrypt/live/example.com/chain.pem;

    include snippets/ssl-params.conf;
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;

    server_name example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_trusted_certificate /etc/letsencrypt/live/example.com/chain.pem;

    include snippets/ssl-params.conf;
    location / {
            proxy_pass http://localhost:3000;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto https;
        }
}
```

Перевіряємо конфіг у nginx

```
sudo nginx -t
```

Рестартуємо nginx

```
sudo systemctl restart nginx
```

## Додавання DNS records

Щоб підключити наш домен, нам потрібно добавити такі записи

Type: A; Name: @ чи www Value: 00.000.0.00 - ( Ваша IP адреса серверу ) Записи DNS обновляються деякий час, тому зміни
запрацюють не одразу.

Перевіряємо сайт та радуємось

```
curl example.com
```

## Автоматичне поновлення SSL сертифікатів

Автоматичне поновлення SSL-сертифікатів з Nginx можна налаштувати з використанням Certbot та системи автоматичного
планування задач cron. Цей пункт передбачає, що у вас уже встановлено certbot python3-certbot-nginx та уже добавлено
сертифкати на ваш сервер.

Додайте новий файл в папку /etc/cron.weekly/ з назвою, наприклад, certbot-renew:

```
sudo nano /etc/cron.weekly/certbot-renew
```

Додайте наступний скрипт у створений файл та збережіть його:

```
#!/bin/sh
/usr/bin/certbot renew --quiet --renew-hook "/usr/sbin/service nginx reload"
```

Цей скрипт автоматично виконує оновлення сертифікатів Certbot та перезавантажує сервер Nginx.

Зробіть файл виконуваним:

```
sudo chmod +x /etc/cron.weekly/certbot-renew
```

Тепер SSL-сертифікати будуть автоматично поновлюватись щотижня. Ви можете налаштувати більш часте оновлення, змінивши
час запуску скрипту у файлі /etc/crontab. Також, Certbot встановить повідомлення електронної пошти про будь-які проблеми
поновлення сертифікатів або про те, що SSL-сертифікат майже закінчується та його потрібно оновити вручну.
