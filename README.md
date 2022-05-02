# Памятка-инструкция по установке и настройке Postgres Pro

1. [Исходные данные](#source)  
3. [Справочная информация](#info)  
4. [Выполнение](#exec)  
    - [Задача 1](#task1)  

## Исходные данные <a name="source"></a>
### Состав стенда
| № | Полное имя узла | IP           |ОС                    | ПО                       |
| - |-----------------|--------------|----------------------|--------------------------|
| 1 | node1.pgpro.lan | 192.168.0.11 | Debian 11 (bullseye) | Postgres Pro 14 Standart |
| 2 | node2.pgpro.lan | 192.168.0.12 | Debian 11 (bullseye) | Postgres Pro 14 Standart |
| 3 | pgadmin.lan     | 192.168.0.13 | Debian 11 (bullseye) | pgadmin4                 |
| 4 | ws.lan          | 192.168.0.14 | Debian 11 (bullseye) | psql                     |

### Настройка стенда с использованием LXC-контейнеров в Proxmox VE
Перед созданием LXC-контейнеров генерируем пару ключей.  
```sh
ssh-keygen -t ed25519 -C "db_admin@ws.lan" -N '' -f ~/.ssh/db_admin -q 
```
Создаем контенейры и указываем в настройках публичный ключ `db_admin.pub`.

Для упрощения подключенияу узлам стенда в файл `~/.ssh/config` заносим следующий текст:
```sh
Host node1.pgpro
    HostName 192.168.0.11
    User root
    Port 22
    IdentityFile ~/.ssh/db_admin

Host node2.pgpro
    HostName 192.168.0.12
    User root
    Port 22
    IdentityFile ~/.ssh/db_admin

Host pgadmin
    HostName 192.168.0.13
    User root
    Port 22
    IdentityFile ~/.ssh/db_admin
    
Host ws
    HostName 192.168.0.14
    User root
    Port 22
    IdentityFile ~/.ssh/db_admin    
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



Для серверных узлов node1.pgpro и node2.pgpro:
```shell
apt install -y postgrespro-std-14
```
```shell
su - postgres -c psql
```
Для клиентских узлов:
```shell
apt install -y postgrespro-std-14-client
```
Для того, чтобы вызывать бинарные файлы PostgreSQL без указания пути создадим необходимые символические ссылки:
```shell
/opt/pgpro/std-14/bin/pg-wrapper links update
```

## Настройка СУБД
смотрим где находится конфигурационнфй файл
```shell
su - postgres -c 'psql -c "SHOW config_file;"'
```
```
                config_file                 
--------------------------------------------
 /var/lib/pgpro/std-14/data/postgresql.conf
(1 строка)
```
Файл `/var/lib/pgpro/std-14/data/postgresql.conf`
В директиве  `listen_addresses = '192.168.0.0'` указываем IP сервера, к которому будет доступно внешнее подключение.

Файл `/var/lib/pgpro/std-14/data/pg_hba.conf`
добавляем строку для возможности удаленного подключения от 192.168.0.0 (pgAdmin)
```
host    all             all             192.168.152.201/32      md5
```


```shell
systemctl restart postgrespro-std-14.service
```

Устанавливаем пароль для postgres:
```shell
su - postgres -c 'psql -c "\password postgres"'
```
