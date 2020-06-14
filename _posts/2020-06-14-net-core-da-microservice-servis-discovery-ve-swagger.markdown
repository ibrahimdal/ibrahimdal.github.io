---
layout: post
title:  ".Net Core Micro Servis ve Swagger"
description: 
date:   2020-06-14 22:01:36 +0530
categories: .NetCore MicroService Swagger
---
Bu yazımda [şuradaki](https://ibrahimdal.github.io/posts/net-core-da-microservice-servis-discovery-ornegi/) oluşturduğumuz micro servis yapısına swagger entegre edeceğim. Swagger sayesinde controller'daki metodları kolaylıkla test edebiliriz ve api larımızı dökümante edebiliriz.

Swagger ekleme işlemini sadece Basket servisine ekleyeceğim diğer product ve order servisleri içinde bu işlem aynı olacaktır.

Öncelikle Nuget 'ten Swashbuckle.AspNetCore paketini Basket servisine yüklüyoruz.

Ardından Startup.cs dosyasında ConfigureServices fonksiyonuna aşağıdaki satırları ekliyoruz.

```c#
services.AddSwaggerGen(c =>
            {
                c.SwaggerDoc("v1", new OpenApiInfo { Title = "Basket API", Version = "v1" });
            });
```

Ardından api'ye çalıştığında swagger'ı kullanmasını söylememiz gerekiyor. Configure fonksiyonunda bu işlemi yapıyoruz.

```c#
app.UseSwagger();
app.UseSwaggerUI(c =>
    {
    c.SwaggerEndpoint("/swagger/v1/swagger.json", "My API V1");
    }
);
```

Swagger'ı eklemek bu kadar kolay localhost:PORT/swagger diyerek swagger ile oluşturduğumuz dökümanı görebiliriz.