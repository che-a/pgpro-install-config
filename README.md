# Памятка-инструкция по установке и настройке Postgres Pro

1. [Исходные данные](#source)  
2. [Цель](#target)  
3. [Справочная информация](#info)  
4. [Выполнение](#exec)  
    - [Задача 1](#task1)  
    - [Задача 2](#task2)
    - [Задача 3](#task3) 

## Исходные данные <a name="source"></a>
### Ссылки на ресурсы
### Состав
| № | Полное имя узла   | ОС                   | ПО                       | Роль                                 |
| - |-------------------|----------------------|--------------------------|--------------------------------------|
| 1 | node1.pgpro.lan   | Debian 11 (bullseye) | Postgres Pro 14 Standart | Сервер СУБД                          |
| 2 | node2.pgpro.lan   | Debian 11 (bullseye) | Postgres Pro 14 Standart | Сервер СУБД                          |
| 3 | pgadmin.lan       | Debian 11 (bullseye) | pgadmin4                 | Серверное приложение для работы с БД |
| 4 | ws.lan            | Debian 11 (bullseye) | psql                     | Рабочая станция                      |

### Цель <a name="target"></a>
- Установить два экземпляра СУБД Postgres Pro на разных узлах.  
- Установить приложение для работы с БД pgadmin4.  
- Настроить шифрованное соедиение клиента с сервером.

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


