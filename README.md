# Установка и настройка Postgres Pro

1. [Исходные данные](#source)  
3. [Справочная информация](#info)  
4. [Выполнение](#exec)  
    - [Задача 1](#task1)  

## Исходные данные <a name="source"></a>
### Ссылки на ресурсы
- [Защита соединений TCP/IP с применением SSL](https://postgrespro.ru/docs/postgrespro/14/ssl-tcp)
- [Поддержка SSL](https://postgrespro.ru/docs/postgrespro/14/libpq-ssl)
- [Файл pg_hba.conf](https://postgrespro.ru/docs/postgrespro/14/auth-pg-hba-conf)
- [Установка и настройка pgAdmin 4 в режиме сервера](https://interface31.ru/tech_it/2021/01/ustanovka-i-nastroyka-pgadmin-4-v-rezhime-servera.html)
- [Установка Postgres Pro 10 для 1С:Предприятие на Debian / Ubuntu](https://interface31.ru/tech_it/2018/10/ustanovka-postgresql-10-dlya-1spredpriyatie-na-debian-ubuntu.html)
- [ Настройка PostgreSQL для работы с клиентами через SSL](http://www.zaweel.ru/2016/08/postgresql-ssl.html)

### Состав стенда
| № | Полное имя  | IP     |              |ОС                    | ПО                       |
| - |-------------|--------|--------------|----------------------|--------------------------|
| 1 | pgpro.lan   | сервер | 192.168.0.11 | Debian 11 (bullseye) | Postgres Pro 14 Standart |
| 2 | pgadmin.lan | клиент | 192.168.0.12 | Debian 11 (bullseye) | pgadmin4                 |
| 3 | psql.lan    | клиент | 192.168.0.13 | Debian 11 (bullseye) | psql                     |

### Настройка стенда с использованием LXC-контейнеров в Proxmox VE
Перед созданием LXC-контейнеров генерируем пару ключей.  
```sh
ssh-keygen -t ed25519 -C "training_admin@pve.lan" -N '' -f ~/.ssh/training_admin -q 
```
Создаем контенейры и указываем в настройках публичный ключ `training_admin.pub`.

Для упрощения подключенияу узлам стенда в файл `~/.ssh/config` заносим следующий текст:
```sh
Host pgpro
    HostName 192.168.0.11
    User root
    Port 22
    IdentityFile ~/.ssh/training_admin

Host pgadmin
    HostName 192.168.0.12
    User root
    Port 22
    IdentityFile ~/.ssh/training_admin
    
Host psql
    HostName 192.168.0.13
    User root
    Port 22
    IdentityFile ~/.ssh/training_admin    
```
### Первичная настройка LXC-контейнера (вариант)
```sh
# Настройка приглашения командной строки:
(
    echo '### New command prompt:';
    echo 'PS1="${debian_chroot:+($debian_chroot)}\[\033[0;31m\]\u\[\033[0;36m\]@\[\033[0;33m\]\H\[\033[0;36m\]:\w\[\033[0;36m\]>\[\033[0;31m\] # \[\033[0m\]"';
    echo 'export PS1'    
  ) >> ~/.bashrc
source ~/.bashrc

apt-get update -y && apt-get upgrade -y
apt-get install -y \
    debconf-utils \
    curl \
    htop \
    lsb-release \
    mc \
    tmux \
    tree \
    wget

# Неинтерактивная замена для dpkg-reconfigure locales
echo "locales locales/default_environment_locale select ru_RU.UTF-8" | debconf-set-selections
echo "locales locales/locales_to_be_generated multiselect ru_RU.UTF-8 UTF-8" | debconf-set-selections
rm "/etc/locale.gen"
dpkg-reconfigure --frontend noninteractive locales

# Установка часового пояса:
timedatectl set-timezone Europe/Moscow

reboot
```

## Установка
### Общие действия для всех узлов
#### Подготовка
На каждом узле необходимо выполнить следующий набор команд:
```shell
apt update && \
apt upgrade -y && \
apt install -y gnupg gnupg1 gnupg2
```

#### Настройка репозитория
Для узлов `pgpro` и `psql`:
```shell
wget http://repo.postgrespro.ru/pgpro-14/keys/GPG-KEY-POSTGRESPRO && \
    apt-key add GPG-KEY-POSTGRESPRO

echo deb http://repo.postgrespro.ru/pgpro-14/debian/ bullseye main > /etc/apt/sources.list.d/postgrespro.list
apt update
```

Для узла `pgpro`:
```shell
apt install -y postgrespro-std-14
```
Для узла `psql`:
```shell
apt install -y postgrespro-std-14-client

# Для того, чтобы вызывать бинарные файлы PostgreSQL без указания пути создадим необходимые символические ссылки:
/opt/pgpro/std-14/bin/pg-wrapper links update
```
## Создание пользователя и БД
```shell
su - postgres -c psql
```
```sql
CREATE USER wiki_user WITH NOCREATEDB NOCREATEROLE NOSUPERUSER ENCRYPTED PASSWORD 'password';
CREATE DATABASE wiki_db WITH OWNER wiki_user;

CREATE USER cloud_user WITH NOCREATEDB NOCREATEROLE NOSUPERUSER ENCRYPTED PASSWORD 'password';
CREATE DATABASE cloud_db WITH OWNER cloud_user;

-- DROP DATABASE db;
-- DROP USER user;
```

## Настройка СУБД
### Каталог с файлами настроек
```shell
# Так можно узнать расположение главного конфигурационного файла
su - postgres -c 'psql -c "SHOW config_file;"'
```
```
                config_file                 
--------------------------------------------
 /var/lib/pgpro/std-14/data/postgresql.conf
(1 строка)
```

### Настройка подключения
В файле `/var/lib/pgpro/std-14/data/postgresql.conf` раскомментировать директиву `listen_addresses` и указать IP-адрес сервера, с которого будет доступно внешнее подключение.
```shell
listen_addresses = '192.168.0.11'    # what IP address(es) to listen on; 
# или
# listen_addresses = '*'
```
В файле `/var/lib/pgpro/std-14/data/pg_hba.conf` настраивается подлкючение к СУБД.
```sh
# Нешифрованный доступ к wiki_db
host    wiki_db        wiki_user        192.168.0.12/32      md5
host    wiki_db        wiki_user        192.168.0.13/32      md5

# Шифрованный (TLS) доступ к cloud_db
hostssl cloud_db       cloud_user       192.168.0.12/32      cert
hostssl cloud_db       cloud_user       192.168.0.13/32      cert
```
Если в файле `/var/lib/pgpro/std-14/data/pg_hba.conf` настроено подключение пользователя `postgres`, то необходимо для него установить пароль:
```sh
su - postgres -c 'psql -c "\password postgres"'
```

#### Создание сертификатов
Для организации шифрованного TLS-соединения необходмы сертификаты.
```sh
# Корневой сертификат ЦС
openssl req -new -nodes -text -out root.csr \
  -keyout root.key -subj "/CN=root.lan"
chmod og-rwx root.key
openssl x509 -req -in root.csr -text -days 3650 \
  -extfile /etc/ssl/openssl.cnf -extensions v3_ca \
  -signkey root.key -out root.crt

# Сертификат сервера pgpro
openssl req -new -nodes -text -out server.csr \
  -keyout server.key -subj "/CN=pgpro.lan"
chmod og-rwx server.key
openssl x509 -req -in server.csr -text -days 365 \
  -CA root.crt -CAkey root.key -CAcreateserial \
  -out server.crt
  
# Сертификат клиента (пользователь cloud_user)
openssl req -new -nodes -text -out client.csr \
  -keyout client.key -subj "/CN=cloud_user"
chmod og-rwx client.key
openssl x509 -req -in client.csr -text -days 365 \
  -CA root.crt -CAkey root.key -CAcreateserial \
  -out client.crt  
```
Выходные файлы:  
|  № | Файл             | Назначение                | Место хранения        |
|----|------------------|---------------------------|-----------------------|
|  1 | root.csr         | Запрос на получение серт. |                       |
|  2 | root.key         | Закрытый ключ             | В изолированном месте |
|  3 | root.crt         | Сертификат корневого ЦС   | На клиенте            |
|  4 | root.srl         |                           |                       |
|  5 | server.csr       |                           |                       |
|  6 | server.key       |                           | На сервере            |
|  7 | server.crt       |                           | На сервере            |
|  8 | client.csr       |                           |                       |
|  9 | client.key       |                           |                       |
| 10 | client.crt       |                           |                       |


##### Установка сертификатов на сервер
Файлы сертификатов необходимо поместить в каталог: `/var/lib/pgpro/std-14/data`.  
```sh
scp root.crt server.{key,crt} pgpro:/var/lib/pgpro/std-14/data/
ssh pgpro "chown postgres: /var/lib/pgpro/std-14/data/server.{key,crt} /var/lib/pgpro/std-14/data/root.crt"
ssh pgpro "chmod 600 /var/lib/pgpro/std-14/data/server.crt"
```
Настройка `TLS` в файле `/var/lib/pgpro/std-14/data/postgresql.conf`
```sh
# - SSL -

ssl = on
ssl_ca_file = 'root.crt'
ssl_cert_file = 'server.crt'
ssl_key_file = 'server.key'
ssl_ciphers = 'HIGH:MEDIUM:+3DES:!aNULL' # allowed SSL ciphers
```
```shell
systemctl restart postgrespro-std-14.service
```
##### Установка сертификатов на клиент
```sh
ssh psql "mkdir /root/.postgresql/"

scp root.crt psql:/root/.postgresql/root.crt
scp client.key psql:/root/.postgresql/postgresql.key
scp client.crt psql:/root/.postgresql/postgresql.crt

ssh psql "chmod 600 /root/.postgresql/postgresql.key"
```
```sh
ssh pgadmin "mkdir -p /var/www/.postgresql/"

scp root.crt pgadmin:/var/www/.postgresql/root.crt
scp client.key pgadmin:/var/www/.postgresql/postgresql.key
scp client.crt pgadmin:/var/www/.postgresql/postgresql.crt

ssh pgadmin "chmod 600 /var/www/.postgresql/postgresql.key"
```

#### Подключение к серверу
Удаленное подключение консольного клиента к серверу:
```sh
psql -U user2 -d db1 -h pgpro-1.lan
```

## Настройка клиента
### Установка и настройка pgAdmin4
```sh
wget https://www.pgadmin.org/static/packages_pgadmin_org.pub
apt-key add packages_pgadmin_org.pub

echo "deb https://ftp.postgresql.org/pub/pgadmin/pgadmin4/apt/$(lsb_release -cs) pgadmin4 main" > /etc/apt/sources.list.d/pgadmin4.list

apt update
apt install pgadmin4-web
```
Далее необходимо запустить скрипт начальной настройки сервера:
```sh
/usr/pgadmin4/bin/setup-web.sh
```

#### Настройка HTTPS на веб-сервере
Создание самоподписанного сертификата:
```sh
openssl req -x509 -nodes -days 3650 -newkey rsa:2048 -keyout /etc/ssl/private/pgadm4-selfsigned.key -out /etc/ssl/certs/pgadm4-selfsigned.crt
```
В файле `/etc/apache2/sites-available/default-ssl.conf` изменим строки:
```sh
SSLCertificateFile      /etc/ssl/certs/pgadm4-selfsigned.crt
SSLCertificateKeyFile /etc/ssl/private/pgadm4-selfsigned.key
```
Далее необходимо создать файл `/etc/apache2/conf-available/ssl.conf` и внести в него следующие строки:
```sh
SSLProtocol all -SSLv3 -TLSv1 -TLSv1.1
SSLCipherSuite ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
SSLHonorCipherOrder off
SSLSessionTickets off

SSLUseStapling On
SSLStaplingCache "shmcb:logs/ssl_stapling(32768)"
```
```sh
# Сохраняем изменения и включаем нужные модули, конфигурации и виртуальные хосты:
a2enmod ssl
a2enconf ssl
a2ensite default-ssl

# Проверяем конфигурацию Apache на ошибки:
apachectl -t

# И перезапускаем веб-сервер:
systemctl reload apache2
```

#### Настройка автоматической переадресации с HTTP на HTTPS
Откроем файл `/etc/apache2/sites-available/000-default.conf` и в пределах секции `VirtualHost` внесем следующие строки:
```sh
RewriteEngine On
RewriteCond %{HTTPS} off
RewriteRule (.*) https://%{HTTP_HOST}%{REQUEST_URI}
```
```sh
# Подключим необходимые модули:
a2enmod rewrite

# И перезапустим веб-сервер:
systemctl reload apache2
```
