---
layout: post
title:  "Postgre Sql'i Docker Üzerinde Ayağa Kaldırma"
description: 
date:   2020-06-06 16:28:36 +0530
categories: Docker PostgreSql
---
PostgreSql'in docker üzerinde kurulumuna kısaca değineceğiz. 

Öncelikle [docker hub](https://hub.docker.com/_/microsoft-mssql-server) daki PostgreSql imajını localimize çekmeliyiz. Bunu aşağıdaki komut ile yapıyoruz.

```shell
docker pull postgres
```

Ardında aşağıdaki docker-compose.yml dosyasını herhangi bir dizin içine oluşturalım. Örn: E:\postgres

```dockerfile
version: "3.7"

services:
    db:
        image: postgres
        environment:
            POSTGRES_PASSWORD: 123456
            PGDATA: /var/lib/postgresql/data/pgdata
        ports: 
            - 5432:5432
        volumes:
            - postgresv:/var/lib/postgresql/data
volumes:
    postgresv:
```

Burada postgres imajından bir servis (db) oluşturuyoruz. Bu servisin port maping işlemini, environment değerlerini ve volume işlemlerini gerçekleştiriyoruz. Volume'un sebebi docker container'ını kapattıktan sonrada datalarımızın kaybolmaması.

Ardında dosyanın bulunduğu dizinde *docker-compose up* diyerek docker üzerinde çalışan bir Postgre elde etmiş oluyoruz. 

DBeaver kullanarak postgres 123456 bilgileri ile postgre üzerinde işlemlerimizi gerçekleştirebiliriz.