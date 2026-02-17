# Отчет по лабораторной работе №1
## Тема: HA Postgres Cluster (Patroni + ZooKeeper + HAProxy)

### Ход работы

### Подготовка файлов конфигурации

Dockerfile

```Dockerfile
FROM postgres:15

RUN apt-get update -y && \
    apt-get install -y netcat-openbsd python3-pip curl python3-psycopg2 python3-venv iputils-ping

RUN python3 -m venv /opt/patroni-venv && \
    /opt/patroni-venv/bin/pip install --upgrade pip && \
    /opt/patroni-venv/bin/pip install patroni[zookeeper] psycopg2-binary

COPY postgres0.yml /postgres0.yml
COPY postgres1.yml /postgres1.yml

ENV PATH="/opt/patroni-venv/bin:$PATH"

USER postgres

```

postgres0.yml

```yml
scope: my_cluster
name: postgresql0

restapi:
  listen: 0.0.0.0:8008
  connect_address: pg-master:8008

zookeeper:
  hosts: ['zoo:2181']

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    master_start_timeout: 300
    synchronous_mode: true
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        wal_level: replica
        hot_standby: "on"
        wal_keep_segments: 8
        max_wal_senders: 10
        max_replication_slots: 10
        wal_log_hints: "on"
        archive_mode: "always"
        archive_timeout: 1800s
        archive_command: mkdir -p /tmp/wal_archive && test ! -f /tmp/wal_archive/%f && cp %p /tmp/wal_archive/%f

  pg_hba:
    - host replication replicator 0.0.0.0/0 md5
    - host all all 0.0.0.0/0 md5

postgresql:
  listen: 0.0.0.0:5432
  connect_address: pg-master:5432
  data_dir: /var/lib/postgresql/data/postgresql0
  bin_dir: /usr/lib/postgresql/15/bin
  pgpass: /tmp/pgpass0
  authentication:
    replication:
      username: replicator
      password: rep-pass
    superuser:
      username: postgres
      password: postgres
  parameters:
    unix_socket_directories: '.'

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false
```

postgres1.yml

```yml
scope: my_cluster
name: postgresql1

restapi:
  listen: 0.0.0.0:8008
  connect_address: pg-slave:8008

zookeeper:
  hosts: ['zoo:2181']

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    master_start_timeout: 300
    synchronous_mode: true
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        wal_level: replica
        hot_standby: "on"
        wal_keep_segments: 8
        max_wal_senders: 10
        max_replication_slots: 10
        wal_log_hints: "on"
        archive_mode: "always"
        archive_timeout: 1800s
        archive_command: mkdir -p /tmp/wal_archive && test ! -f /tmp/wal_archive/%f && cp %p /tmp/wal_archive/%f

  pg_hba:
    - host replication replicator 0.0.0.0/0 md5
    - host all all 0.0.0.0/0 md5

postgresql:
  listen: 0.0.0.0:5432
  connect_address: pg-slave:5432
  data_dir: /var/lib/postgresql/data/postgresql1
  bin_dir: /usr/lib/postgresql/15/bin
  pgpass: /tmp/pgpass0
  authentication:
    replication:
      username: replicator
      password: rep-pass
    superuser:
      username: postgres
      password: postgres
  parameters:
    unix_socket_directories: '.'

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false
```

haproxy.cfg

```cfg
global
    maxconn 100

defaults
    log global
    mode tcp
    retries 2
    timeout client 30m
    timeout connect 4s
    timeout server 30m
    timeout check 5s

listen stats
    mode http
    bind *:7000
    stats enable
    stats uri /

listen postgres
    bind *:5432
    option httpchk
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server pg-master pg-master:5432 maxconn 100 check port 8008
    server pg-slave pg-slave:5432 maxconn 100 check port 8008

```

docker-compose.yml

```yml
services:
  zoo:
    image: confluentinc/cp-zookeeper:7.7.1
    container_name: zoo
    restart: always
    hostname: zoo
    ports:
      - 2181:2181
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000

  pg-master:
    build: .
    image: my-postgres-patroni
    container_name: pg-master
    hostname: pg-master
    restart: always
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      PATRONI_NAME: postgresql0
    ports:
      - "5433:5432"
      - "8008:8008"
    command: patroni /postgres0.yml
    depends_on:
      - zoo

  pg-slave:
    build: .
    image: my-postgres-patroni
    container_name: pg-slave
    hostname: pg-slave
    restart: always
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      PATRONI_NAME: postgresql1
    ports:
      - "5434:5432"
      - "8009:8008"
    command: patroni /postgres1.yml
    depends_on:
      - zoo
      - pg-master

  haproxy:
    image: haproxy:2.4
    container_name: haproxy
    ports:
      - "5000:5432"
      - "7000:7000"
    volumes:
      - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro
    depends_on:
      - pg-master
      - pg-slave

```

### Запускаем compose файл

![alt text](<imgs/Screenshot_68.png>)

### Проверяем логи, убеждаемся что postgres1 - seconadary

![alt text](<imgs/Screenshot_69.png>)

### Подключаемся к master и slave в pgAdmin 4

![alt text](<imgs/Screenshot_70.png>)

### Создаём таблицу в master и добавляем туда данные

![alt text](<imgs/Screenshot_71.png>)

### Убеждаемся воочию, что таблица создалась

![alt text](<imgs/Screenshot_72.png>)

### Смотрим, что в slave появились те же данные

![alt text](<imgs/Screenshot_73.png>)

### Убеждаемся, что slave работает в режиме read-only

![alt text](<imgs/Screenshot_74.png>)

### Смотрим логи haproxy, видим, что подключились к master базе

![alt text](<imgs/Screenshot_75.png>)

### Принудительно кладём контейнер с slave и опять смотрим логи haproxy.

![alt text](<imgs/Screenshot_77.png>)

![alt text](<imgs/Screenshot_76.png>)

### Добавляем данные через прокси

![alt text](<imgs/Screenshot_78.png>)

### Видим, что они появились в мастере

![alt text](<imgs/Screenshot_79.png>)

### При этом slave мы не можем открыть, так как он отключен

![alt text](<imgs/Screenshot_80.png>)

### Сотрим логи, видим, что прокси пытался обратиться к обоим серверам, но смог добавить только в master

![alt text](<imgs/Screenshot_81.png>)

### Ответы на вопросы

1. При обычном перезапуске композ-проекта, будет ли сбилден заново образ? Нет. Docker использует кэш. Если файлы не менялись и не передан флаг --build, будет использован локальный образ.

2. А если предварительно отредактировать файлы postgresX.yml? Нет. Эти файлы копируются внутрь образа при сборке. Изменение их на хосте не обновит их внутри образа без пересборки.

3. А если содержимое самого Dockerfile? Тоже нет, пока не будет запущена команда сборки. Docker Compose проверяет наличие образа, а не хеш Dockerfile по умолчанию при простом старте (если образ уже есть).

4. Порты 8008 и 5432 вынесены в разные директивы, expose и ports. В чем разница?
ports (например, 5433:5432) публикует порт на хост-машину (доступен из Windows).
expose (например, 8008) открывает порт только внутри виртуальной сети Docker для взаимодействия между контейнерами (например, чтобы HAProxy видел API Patroni), но не открывает его наружу.
