# Памятка-инструкция по установке и настройке Postgres Pro

1. [Исходные данные](#source)  
3. [Справочная информация](#info)  
4. [Выполнение](#exec)  
    - [Задача 1](#task1)  

## Исходные данные <a name="source"></a>
### Состав стенда
| № | Полное имя узла | IP           |ОС                    | ПО                       |
| - |-----------------|--------------|----------------------|--------------------------|
| 1 | pgpro-1.lan     | 192.168.0.11 | Debian 11 (bullseye) | Postgres Pro 14 Standart |
| 2 | pgpro-2.lan     | 192.168.0.12 | Debian 11 (bullseye) | Postgres Pro 14 Standart |
| 3 | pgadmin.lan     | 192.168.0.13 | Debian 11 (bullseye) | pgadmin4                 |
| 4 | ws.lan          | 192.168.0.14 | Debian 11 (bullseye) | psql                     |

### Настройка стенда с использованием LXC-контейнеров в Proxmox VE
Перед созданием LXC-контейнеров генерируем пару ключей.  
```sh
ssh-keygen -t ed25519 -C "training_admin@ws.lan" -N '' -f ~/.ssh/training_admin -q 
```
Создаем контенейры и указываем в настройках публичный ключ `training_admin.pub`.

Для упрощения подключенияу узлам стенда в файл `~/.ssh/config` заносим следующий текст:
```sh
Host pgpro-1
    HostName 192.168.0.11
    User root
    Port 22
    IdentityFile ~/.ssh/training_admin

Host pgpro-2
    HostName 192.168.0.12
    User root
    Port 22
    IdentityFile ~/.ssh/training_admin

Host pgadmin
    HostName 192.168.0.13
    User root
    Port 22
    IdentityFile ~/.ssh/training_admin
    
Host ws
    HostName 192.168.0.14
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
Для всех узлов:
```shell
wget http://repo.postgrespro.ru/pgpro-14/keys/GPG-KEY-POSTGRESPRO && \
    apt-key add GPG-KEY-POSTGRESPRO

echo deb http://repo.postgrespro.ru/pgpro-14/debian/ bullseye main > /etc/apt/sources.list.d/postgrespro.list
apt update
```

Для серверных узлов:
```shell
apt install -y postgrespro-std-14
```
Для клиентских узлов:
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

В файле `/var/lib/pgpro/std-14/data/pg_hba.conf` настраивается подключение к СУБД:
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
Если не настраивать шифрованное TLS-соединие между сервером и клиентом, то в файл `/var/lib/pgpro/std-14/data/pg_hba.conf` добавляются следующие строки:
```sh
# Доступ к db1 с узлов pgadmin и ws
host db1             user1           192.168.0.13/32      md5
host db1             user1           192.168.0.14/32      md5

# Доступ к db2 с узлов pgadmin и ws
host db2             user2           192.168.0.13/32      md5
host db2             user2           192.168.0.14/32      md5
```

Чтобы иметь возможность подключаться как `postgres`, необходимо для него установить пароль:
```sh
su - postgres -c 'psql -c "\password postgres"'
```
