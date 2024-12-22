

# Задача
В этом разделе описан процесс миграции основной базы данных платформы, которая работает в контейнере в иную среду - в физический или виртуальный сервер.

<img src="move_db_1.png">

## Назначение
Существует множество причин при которых база данных должна функционировать не в контейнере.
Например:
* Ограничения по ресурсам.
* Безопасность данных.
* Сетевые ограничения.
* Особенности инфраструктуры.
* Прочие причины.

Как результат данной операции основная база данных переносится из контейнерной среды функционирования непосредственно в среду операционной системы без использования дополнительного слоя абстракции.

## Описание процесса

Перенос базы данных состоит их следующих этапов:
1. Сбор требований.
2. Подготовка целевой среды.<br/>
   Задача данного этапа подготовить целевую среду исполнения основной базы данных Платформы. Это включает в себя установку необходимого программного обеспечения и его настройку. Данный этап может быть пропущен если целевая среда уже существует и настроена.
3. Экспорт файлов базы данных во временный архивный файл.
4. Конфигурация Postgres на использование копии данных.
5. Изменение настроек платформы.
6. Запуск платформы и заключительные проверки.


### Важно

1. Миграция базы данных желательно должна осуществляться на ту же версию Postgres которая функционирует в контейнере.
В случае если рассматривается миграция на другую версию или тип Postgres, необходимо связаться с производителем/поставщиком платформы для консультаций.

2. Документ описывает процесс миграции базы данных для версии платформы **VVV**.

### Обозначения

- **PG_VERSION** означает номер версии Postgres на которой функционирует Платформа.


# Этапы миграции
## Сбор требований

До начала основного процесс миграции необходимо собрать следующую информацию:
1. Выделение необходимого дискового пространства под базу данных.
2. Версия базы данных Postgres используемой Платформой. Разные версии платформы могут функционировать с разными версиями Postgres.
3. Настройка сетевого доступа из docker сети до новой среды функционирования основной базы данных.



## Подготовка целевой среды

Необходимо установить репозиторий Postgres для правильной установки целевой версии Postgres.

Алгоритм подготовки представляет следующие шаги (представленные ниже команды были выполнены на ОС Astra Linux SE 1.7.3 «Смоленск»/»Орел»):

1. Добавьте ключ и укажите репозиторий Tantor. Замените USERNAME и PASSWORD на необходимые логин и пароль, предоставленные компанией «Лаборатории Тантор».

```bash
wget --quiet -O - https://public.tantorlabs.ru/tantorlabs.ru.asc | apt-key add -
```

```bash
echo "deb [arch=amd64] https://USERNAME:PASSWORD@nexus.tantorlabs.ru/repository/tantorlabs-apt smolensk main" > /etc/apt/sources.list.d/tantorlabs.list
```

2. Измените приоритет репозиториев
```bash
echo "Package: *
Pin: origin nexus.tantorlabs.ru
Pin-Priority: 1001

Package: *
Pin: release n=1.7_x86-64
Pin-Priority: 900" > /etc/apt/preferences.d/smolensk
```


3. Установка репозиториев Astra Linux (опционально)
*Внимание: выполняется если репозитории Astra Linux не установлены*
```bash
echo "# Astra Linux repository description https://wiki.astralinux.ru/x/0oLiC

deb cdrom:[OS Astra Linux 1.7.3 1.7_x86-64 DVD ]/ 1.7_x86-64 contrib main non-free
deb https://download.astralinux.ru/astra/stable/1.7_x86-64/repository-main/ 1.7_x86-64 main contrib non-free
deb https://download.astralinux.ru/astra/stable/1.7_x86-64/repository-update/ 1.7_x86-64 main contrib non-free

deb https://download.astralinux.ru/astra/stable/1.7_x86-64/repository-base/ 1.7_x86-64 main contrib non-free
deb https://download.astralinux.ru/astra/stable/1.7_x86-64/repository-extended/ 1.7_x86-64 main contrib non-free" > /etc/apt/sources.list

apt-get update
```

Добавьте локаль EN_US.UTF-8:
```bash
dpkg-reconfigure locales
```

Найдите в списке EN_US.UTF-8 и подтвердите выбор. Например, двойным нажатием "ОK".
Установите Postgresql необходимой версии/типа:

```bash
apt install postgresql-11 postgresql-11-cron -y
```

Установите расширения:
```bash
apt install pg-framework pg-store-plans-11 pipelinedb pgsql-http -y
```

Активация автозапуска Postgresql:
```bash
systemctl enable postgresql
```

Остановите работу Postgresql:
```bash
systemctl stop postgresql
```
Примечание: Это необходимо так как на следующих этапах Postgres будет реконфигурирован.

## Экспорт файлов базы данных во временный архивный файл

Команды, представленные ниже, выполняются на узле, где установлена Платформа:

Имеет смысл остановить работу основных элементов Платформы которые производят манипуляции с данными:

```bash
docker-compose keeper backend 
```

Доступ в контейнер базы данных в режиме консоли.

Зайдите в docker-контейнер, в котором находится база данных, и выполните команды.
```bash
docker exec -it db bash
```

Добавьте строку для возможности снятия резервной копии базы данных и обновите настройки:

```bash
echo 'local replication all trust' >> /var/lib/postgresql/PG_VERSION/main/pg_hba.conf

su - postgres

/usr/lib/postgresql/PG_VERSION/bin/pg_ctl reload -D /var/lib/postgresql/PG_VERSION/main
```

Создание резервной копию БД:

```bash
pg_basebackup -U postgres -D - -Ft -X fetch | gzip > /tmp/backup.tar.gz
```

Далее необходимо отредактировать файл docker-compose.yml. 
Необходимо внести изменения в следующие строки ручным способом либо автоматически. Команды также представлены ниже.

Было:

```yaml
PMA_SERVER_OPSPG_HOST=db

PMA_KEEPER_OPSPG_HOST=db

POSTGRES_HOST=db

```

Стало:
```yaml
PMA_SERVER_OPSPG_HOST=X.X.X.X

PMA_KEEPER_OPSPG_HOST=X.X.X.X

POSTGRES_HOST=X.X.X.X
```

В командах этого шага X.X.X.X – это IP-адрес машины с будущей отдельной базой, например: *192.168.1.84*
Ниже представлен пример:

```bash
sed -i -e "s/PMA_SERVER_OPSPG_HOST=db/PMA_SERVER_OPSPG_HOST=192.168.1.84/" "/opt/tantor/eco/docker-compose.yml"

sed -i -e "s/PMA_KEEPER_OPSPG_HOST=db/PMA_KEEPER_OPSPG_HOST=192.168.1.84/" "/opt/tantor/eco/docker-compose.yml"

sed -i -e "s/POSTGRES_HOST=db/POSTGRES_HOST=192.168.1.84/" "/opt/tantor/eco/docker-compose.yml"
```



Выйдите из консольной сессии контейнера:
```bash
exit
```

Копирование файла с данными во временную папку Хоста:

```bash
docker cp db:/tmp/backup.tar.gz /tmp
```
Внимание: целевой путь /tmp указан в качестве примера, в зависимости от задачи может использоваться другой путь, а также другая целевая среда. 


Остановите контейнер базы данных:
```bash
docker-compose stop db
``` 

### Исключение контейнера основной базы данных из docker-compose.yaml

Для предотвращения дальнейшего запуска контейнера основной базы данных необходимо отредактировать файл docker-compose.yml.

Удалите или закомменитируйте всю секцию db в файле.


## Конфигурация Postgres на использование копии данных

Удаление файлов данных созданных по умолчанию
```bash
rm -rf /var/lib/postgresql/PG_VERSION/main/*
```

Распаковка копии данных скопированных с контейнера
```bash
tar xvfz /opt/backup.tar.gz -C /var/lib/postgresql/PG_VERSION/main/

```
Перенесите настройки на машину:
```bash
cp /var/lib/postgresql/PG_VERSION/main/pg_hba.conf /etc/postgresql/PG_VERSION/main
cp /var/lib/postgresql/PG_VERSION/main/postgresql.conf /etc/postgresql/PG_VERSION/main
```

Изменение настройек Postgres в файле postgresql.conf:
```bash
sed -i -e "s/var\/lib\/postgresql\/PG_VERSION\/main\/pg_hba.conf/etc\/postgresql\/PG_VERSION\/main\/pg_hba.conf/" "/etc/postgresql/PG_VERSION/main/postgresql.conf"
sed -i -e "s/var\/lib\/postgresql\/PG_VERSION\/main\/pg_ident.conf/etc\/postgresql\/PG_VERSION\/main\/pg_ident.conf/" "/etc/postgresql/PG_VERSION/main/postgresql.conf"
```

Отредактируйте файл pg_hba.conf (/etc/postgresql/PG_VERSION/main/pg_hba.conf), где Y.Y.Y.Y - IP-адрес Платформы:
```bash
echo "local   all             all                                     trust
host    all             all             127.0.0.1/32            md5
host    pma             pmm             Y.Y.Y.Y/32        trust
host    all             all             samenet                 md5
host   all             all             0.0.0.0/0               reject" > /etc/postgresql/PG_VERSION/main/pg_hba.conf
```

Запуск PostgreSQL:
```bash
systemctl start postgresql
```

 Проверить, что Postgres работает, можно например с помощью команды:
```bash
systemctl status postgresql
```
А для проверки работы кластера используйте:
```bash
pg_lsclusters
```

Внесите необходимые изменения в настройки:

```bash
su - postgres
psql -U postgres pma
```
Обновите значение IP-адреса, подставив вместо Y.Y.Y.Y IP-адрес Платформы:
```bash
update sys.base_config set value='Y.Y.Y.Y:5666' where name='backend_rest';
```

## Запуск платформы и заключительные проверки

Далее следует вернуться к Платформе (в ее среду исполнения):

Запустите Платформу:

```bash
cd /opt/tantor/eco/

docker-compose --env-file=platform.env up -d
```

Проверьте статусы контейнеров и убедитесь что все сервисы работоспособны:
```bash
docker-compose ps
```

Проверьте, что Платформа работает с новой базой данных.
Например, сделать это можно осуществив вход в пользовательский интерфейс Платформы и просмотрев общее состояние Лицензии или просмотрев статус мониторинга любого экземпляра Postgres.

