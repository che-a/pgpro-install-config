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
CREATE USER user1 WITH NOCREATEDB NOCREATEROLE NOSUPERUSER ENCRYPTED PASSWORD 'password1';
CREATE DATABASE db1 WITH OWNER user1;

CREATE USER user2 WITH NOCREATEDB NOCREATEROLE NOSUPERUSER ENCRYPTED PASSWORD 'password2';
CREATE DATABASE db2 WITH OWNER user2;

-- DROP DATABASE db1;
-- DROP USER user1;
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
В файле `/var/lib/pgpro/std-14/data/postgresql.conf` в директиве `listen_addresses` необходимо указать IP-адрес этого же узла, к которому будет доступно внешнее подключение.
```shell
# Для pgpro-1
listen_addresses = '192.168.0.11'    # what IP address(es) to listen on; 

# Для pgpro-2
listen_addresses = '192.168.0.12'    # what IP address(es) to listen on; 
```
#### Настройка нешифрованного подключения
Для `pgpro-1` настроим обычное нешифрованное соединение.
Если не настраивать шифрованное TLS-соединие между сервером и клиентом, то в файл `/var/lib/pgpro/std-14/data/pg_hba.conf` добавляются следующие строки:
```sh
# Доступ к db1 с узлов pgadmin и ws
host db1             user1           192.168.0.13/32      md5
host db1             user1           192.168.0.14/32      md5

# Доступ к db2 с узлов pgadmin и ws
host db2             user2           192.168.0.13/32      md5
host db2             user2           192.168.0.14/32      md5
```
```shell
systemctl restart postgrespro-std-14.service
```
Если в файле `/var/lib/pgpro/std-14/data/pg_hba.conf` настроено подключение пользователя `postgres`, то необходимо для него установить пароль:
```sh
su - postgres -c 'psql -c "\password postgres"'
```

#### Настройка зашифрованного TLS-соединения
Для `pgpro-2` настроим шифрованное TLS-соединение.

##### Создание сертификатов
```sh
# корневой сертификат
openssl req -new -nodes -text -out root.csr \
  -keyout root.key -subj "/CN=root.lan"
chmod og-rwx root.key
openssl x509 -req -in root.csr -text -days 3650 \
  -extfile /etc/ssl/openssl.cnf -extensions v3_ca \
  -signkey root.key -out root.crt

# Сертификат сервера
openssl req -new -nodes -text -out server.csr \
  -keyout server.key -subj "/CN=pgpro-2.lan"
chmod og-rwx server.key
openssl x509 -req -in server.csr -text -days 365 \
  -CA root.crt -CAkey root.key -CAcreateserial \
  -out server.crt
  
# Сертификат клиента (пользователь user1)
openssl req -new -nodes -text -out client.csr \
  -keyout client.key -subj "/CN=user1"
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
|  5 | intermediate.csr |                           |                       |
|  6 | intermediate.key |                           | В изолированном месте |
|  7 | intermediate.crt |                           | На сервере            |
|  8 | intermediate.srl |                           |                       |
|  9 | server.csr       |                           |                       |
| 10 | server.key       |                           | На сервере            |
| 11 | server.crt       |                           | На сервере            |

##### Установка сертификатов на сервер
Файлы сертификатов необходимо поместить в каталог: `/var/lib/pgpro/std-14/data`.  
```sh
scp root.crt server.{key,crt} pgpro-2:/var/lib/pgpro/std-14/data/
ssh pgpro-2 "chown postgres: /var/lib/pgpro/std-14/data/server.{key,crt} /var/lib/pgpro/std-14/data/root.crt"
ssh pgpro-2 "chmod 600 /var/lib/pgpro/std-14/data/server.crt"
```
Настройка `TLS` в файле `/var/lib/pgpro/std-14/data/postgresql.conf`
```sh
# - SSL -

ssl = on
ssl_ca_file = 'root.crt'
ssl_cert_file = 'server.crt'
#ssl_crl_file = ''
#ssl_crl_dir = ''
ssl_key_file = 'server.key'
ssl_ciphers = 'HIGH:MEDIUM:+3DES:!aNULL' # allowed SSL ciphers
#ssl_prefer_server_ciphers = on
#ssl_ecdh_curve = 'prime256v1'
#ssl_min_protocol_version = 'TLSv1.2'
#ssl_max_protocol_version = ''
#ssl_dh_params_file = ''
#ssl_passphrase_command = ''
#ssl_passphrase_command_supports_reload = off
```
```shell
systemctl restart postgrespro-std-14.service
```
##### Установка сертификатов на клиент
```sh
ssh ws "mkdir /root/.postgresql/"

scp root.crt ws:/root/.postgresql/root.crt
scp client.key ws:/root/.postgresql/postgresql.key
scp client.crt ws:/root/.postgresql/postgresql.crt

ssh ws "chmod 600 /root/.postgresql/postgresql.key"
```
```sh
ssh pgadmin "mkdir /root/.postgresql/"

scp root.crt pgadmin:/root/.postgresql/root.crt
scp client.key pgadmin:/root/.postgresql/postgresql.key
scp client.crt pgadmin:/root/.postgresql/postgresql.crt

ssh pgadmin "chmod 600 /root/.postgresql/postgresql.key"
```

##### Раздел
В файле `/var/lib/pgpro/std-14/data/pg_hba.conf` узла `pgpro-2` настраивается подключение к СУБД:
```sh
# Доступ к db1 с узлов pgadmin и ws
hostssl db1             user1           192.168.0.13/32      cert
hostssl db1             user1           192.168.0.14/32      cert

# Доступ к db2 с узлов pgadmin и ws
hostssl db2             user2           192.168.0.13/32      cert
hostssl db2             user2           192.168.0.14/32      cert
```

```shell
systemctl restart postgrespro-std-14.service
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
