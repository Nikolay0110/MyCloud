# Backend
## Развёртывание на сервере Linux Ubuntu
1. Для удобного управления из своего терминала, скопируйте публичную часть ssh ключа на сервер:  
```bash
cat ~/.ssh/id_rsa.pub
```
- Укажите ключ в соответствующем поле при создании сервера
- Войдите через консоль на сервер:
- где вместо 0.0.0.0 вводим ip созданного сервера.
```bash
ssh root@0.0.0.0
```
------------------------------------------------------------------------
2. Создайте нового пользователя
- Вместо user_user вводим имя нового пользователя
- И наделите нового пользователя правами `superuser`
```bash
adduser user_name
usermod user_name -aG sudo
```
- Переключитесь на вновь созданного пользователя:
```bash
sudo -i -u user_name
```
------------------------------------------------------------------------
3. Перед установкой обновите список репозиториев и обновите пакеты:
```bash
sudo apt update -y && apt upgrade -y
```
- Устанавливаем следующие пакеты:
```bash
sudo apt-get install python3 python3-venv python3-pip postgresql nginx
```
- Смените пароль пользователя postgres и создайте базу данных
```bash
sudo su postgres
psql
ALTER USER postgres WITH PASSWORD '2312';
CREATE DATABASE my_cloud;
\q
exit
```
- Склонируйте в корень папки вашего пользователя репозиторий с проектом,  
настройте виртуальное окружение Python и установите пакеты из requirements.txt
```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```
- Сделайте миграции и выгрузите данные из заранее подготовленного файла loaddata.json:
```bash
python manage.py migrate
python manage.py loaddata loaddata.json
```
--------------------------------------------------------------------
4. Настройте gunicorn:  
```bash
sudo nano /etc/systemd/system/gunicorn.service
```
- Впишите следующий код, где смените `nikolay` на вашего пользователя
```bash
[Unit]
Description=gunicorn
After=network.target

[Service]
User=nikolay
WorkingDirectory=/home/nikolay/My_cloud/backend
ExecStart=/home/nikolay/My_cloud/backend/venv/bin/gunicorn --access-logfile -\
    --workers=3 \
    --bind unix:/home/nikolay/My_cloud/backend/backend/project.sock backend.wsgi:application

[Install]
WantedBy=multi-user.target

```
Запустите gunicorn, добавьте в автозагрузку и проверьте его работу 
```bash
sudo systemctl start gunicorn
sudo systemctl enable gunicorn
sudo systemctl status gunicorn
```
--------------------------------------------------------------------------
5. Настройте nginx.  
Создадим новый файл конфигурации:
```bash
sudo nano /etc/nginx/sites-available/mycloud
```
Впишите следующий код, смените имя пользователя системы и ip сервера
```bash
server {
    listen 80;
    server_name 194.67.88.152;
    root /home/nikolay/My_cloud/frontend/build;

    location /media/ {
        alias /home/nikolay/My_cloud/backend/backend/media/;
        default_type "image/jpg";
    }
    location / {
        try_files $uri /index.html =404;
    }
    location /api/ {
        include proxy_params;
        proxy_pass http://unix:/home/nikolay/My_cloud/backend/backend/project.sock;
    }
    location /a/ {
        include proxy_params;
        proxy_pass http://unix:/home/nikolay/My_cloud/backend/backend/project.sock;
    }
}
```
- Создайте символическую ссылку на конфиг
```bash
sudo ln -s /etc/nginx/sites-available/mycloud /etc/nginx/sites-enabled
```
- Также вы можете настроить максимальный размер файла для загрузки:
```bash
sudo nano /etc/nginx/nginx.conf
```
- Добавив следующие строки в http{} (в данном примере ограничение 15мб)  
client_max_body_size 15M;  
client_body_buffer_size 15M;

- Переопределите конфиг сервера nginx и проверьте его работоспособность:
```bash
sudo systemctl reload nginx
sudo systemctl status nginx
```
- Логи nginx:
```bash
sudo nano /var/log/nginx/access.log
sudo nano /var/log/nginx/error.log
```
### Теперь ваш сайт должен работать, хорошего пользования)
