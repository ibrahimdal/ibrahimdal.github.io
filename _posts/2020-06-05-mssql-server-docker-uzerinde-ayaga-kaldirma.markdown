---
layout: post
title:  "MsSql Server'ı Docker Üzerinde Ayağa Kaldırma"
description: 
date:   2020-06-05 20:50:36 +0530
categories: Docker MsSqlServer
---
Bu konu kısa olacak, microsoft'un kendisinin bu konuyla alakalı açıklamalarına [buradan](https://docs.microsoft.com/en-us/sql/linux/sql-server-linux-configure-docker?view=sql-server-ver15) ulaşabilirsiniz. Microsoft bu işlemi bash scripti oluşturarak yapılmasını gösteriyor. Ben burada docker-compose ile bu işlemin yapılımından bahsedeceğim.

Öncelikle [docker hub](https://hub.docker.com/_/microsoft-mssql-server) daki Microstf SQL Server imajını localimize çekmeliyiz. Bunu aşağıdaki komut ile yapıyoruz.

```shell
docker pull mcr.microsoft.com/mssql/server
```

Ardında aşağıdaki docker-compose.yml dosyasını herhangi bir dizin içine oluşturalım. Örn: E:\sqlServer

```dockerfile
version: "3.7"

services:
    sqlserver:
        image: mcr.microsoft.com/mssql/server
        ports:
            - 1433:1433
        volumes:
            - sqlvolume:/var/opt/mssql
        environment:
            ACCEPT_EULA: Y
            MSSQL_SA_PASSWORD: 123456_Sql
    
volumes:
    sqlvolume:
```

Burada mcr.microsoft.com/mssql/server imajından bir servis (sqlserver) oluşturuyoruz. Bu servisin port maping işlemini, environment değerlerini ve volume işlemlerini gerçekleştiriyoruz. Volume'un sebebi docker container'ını kapattıktan sonrada datalarımızın kaybolmaması.

Ardında dosyanın bulunduğu dizinde *docker-compose up* diyerek docker üzerinde çalışan bir MsSql Server elde etmiş oluyoruz. 

Sql Management Studio ile . sa 123456_Sql bilgileri ile server üzerinde işlemlerimizi gerçekleştirebiliriz.