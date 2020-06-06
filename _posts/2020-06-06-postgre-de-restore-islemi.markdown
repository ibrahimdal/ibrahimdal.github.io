---
layout: post
title:  "Postgre Sql'de Restore İşlemi"
description: 
date:   2020-06-06 16:35:36 +0530
categories: Docker PostgreSql Restore
---
Postgre'de restore dan kastım, bir veritabanı backup'ını postgre üzerine yüklemek. Bunun için postgre'nin kendi sitesindeki örneği kullanacağız. Ama burada bizim postgre'miz docker üzerinde çalıştığından bu işlemi farklı yolla yapacağız. 

İlk adım olarak [buradaki](https://www.postgresqltutorial.com/postgresql-sample-database/) postgre backup'ını indiriyoruz. Ardında bu backup'ı rardan çıkarıyoruz. Artık elimizde .tar uzantılı bir dosya mevcut. 

Bu dosyayı çalışan postgre container'ına göndermemiz gerekiyor. Aşağıdaki komut ile bunu yapıyoruz.  Buradaki containerId bilgisini öğrenmek için *docker ps* komutunu kullanabilirsiniz.

```shell
docker cp <path>\dvdrental.tar <ContainerId>:dvdrental.tar
```

Artık .tar dosyamız container içinde, bunu pg_restore ile çalıştırmamız gerekiyor. Ama öncesinde *dvdrental* adında bir veritabanı oluşturacağız. Bunun için ben DBeaver aracını kullanacağım. DBeaver 'ı açıyorum, postgre'ye bağlanıyorum ve aşağıdaki sql komutunu sql editöründe çalıştırıyorum.

```sql
CREATE DATABASE dvdrental;
```

postgre container üzerinde interactive moda geçeceğiz, bunu aşağıdaki komut ile yapıyoruz.

```bash
docker exec -it <ContainerId> bash
```

Artık container içindeyiz isterseniz *ls* ile .tar dosyasının orda olup olmadığına bakabiliriz.

```shell
pg_restore -U postgres -d dvdrental dvdrental.tar
```

İşte yukardaki komut ile tar dosyamızı dvdrental veritabanına yüklemiş oluyoruz.

*Not: Tüm bu işlemleri docker ve komut satırı olmadan DBeaver üzerinden de yapabilirsiniz.*