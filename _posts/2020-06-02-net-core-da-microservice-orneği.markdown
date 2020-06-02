---
layout: post
title:  ".Net Core Micro Servis Örneği"
description: 
date:   2020-06-02 21:03:36 +0530
categories: .NetCore C# MicroService
---
Mikro servisler günümüzün trendi, farklı işler yapan, farklı sayıda olan, birbiri ile haberleşebilen, farklı dil ve teknolojilerde olabilen, yatayda genişleyebilen ama tüm resme bakıldığında bir bütün halinde çalışan sistemler.

Bu yazımda .net core ile basit bir mikro servis yapısı nasıl oluyor elimden geldiğince anlatacağım.

Bizim burda bahsedeceğimiz mikro servisler http ile istek alan işleyen ve bir sonuç döndüren servisler olacak. Her zaman verilen örnek, e-ticaret sistemleri üzerinden gidelim. Bir ürün servisimiz olsun ve bir de siparişlerle ilgilenen servisimiz olsun. Ayrıca birde client'ımız olacak.

ProductService : http://localhost:81/api
OrderService : http://localhost:82/api

Yukarıda servislerimi ve adreslerini yazdım. Şimdi client'ıma gidiyorum ve diyorum; ürün işlemleri yaparken şu adrese, sipariş işlemleri yaparken şu adrese git. Bu biraz saçma bir yapı oldu. 

Şöyle olsa, ben http://localhost a git desem ve gelen isteğin sipariş ile mi yoksa ürün ile mi alakalı olduğunu algılasam ve arka tarafta 81 yada 82 ye yönlendirsem. O zaman benim bu yönlendirme işlemini yapacak bir servis oluşturmam gerekiyor.

ApiGateWay : http://localhost:80

Bu şekilde daha derli toplu oldu. O halde bu servisleri .net core ile oluşturalım.

Visual Studio üzerinde SampleMicroService adında bir solution oluşturuyorum. Ardında src, test adında klasör oluşturuyorum. Servisleri src klasörü içine oluşturuyorum.

SampleMicroService.GateWay, SampleMicroService.ProductService, SampleMicroService.OrderService adında .net core api projeleri oluşturuyorum.

SampleMicroService.GateWay isteklerin hangi mikro servise gitmesi gerektiğine karar verecek ve istekleri o servise yönlendirecek. Bu işlemler için 'Ocelot' kütüphanesini kullanacağız. Nuget aracılığıyla Ocelot kütüphanesini Gateway'e yüklüyorum.

GateWay'in Program.cs dosyasını aşağıdaki gibi düzenliyorum.  ConfigureServices içinde Ocelot'u ekliyorum, Configure içinde Ocelot'u kullanmasını söylüyorum. Artık bu servise gelen tüm istekler Ocelot Middleware inden geçecek. Ocelot ise yönlendirme işlemini yapacak, ama bir sorun var bu yönlendirme işlemini nasıl yapacak?

*GateWay.Program.cs
```csharp
public static void Main(string[] args)
{
	new WebHostBuilder()
		.UseKestrel()
		.UseContentRoot(Directory.GetCurrentDirectory())
		.ConfigureAppConfiguration((context, config) =>
		{
			config
				.SetBasePath(context.HostingEnvironment.ContentRootPath)
				.AddJsonFile("appsettings.json", true, true)
				.AddJsonFile(
					$"appsettings.{context.HostingEnvironment.EnvironmentName}.json",
					true,
					true
				)
				.AddJsonFile("ocelot.json")
				.AddEnvironmentVariables();
		})
		.ConfigureServices((context, services) =>
		{
			services.AddOcelot();
		})
		.ConfigureLogging((hostingContext, logging) =>
		{
			logging.AddConsole();
		})
		.Configure(app =>
		{
			app.UseOcelot().Wait();
		})
		.Build()
		.Run();
}
```

Dikkat ettiyseniz yukarıda ocelot.json adında bir dosyayı configurasyon içine gönderiyoruz. Ocelot'da bu dosyadaki belirlemelere göre yönlendirme işlemini gerçekleştiriyor.

GateWay içine ocelot.json adında bir dosya ekliyorum ve içeriğini aşağıdaki gibi düzenliyorum.

*ocelot.json
```json
{
  "Routes": [
    {
      "DownstreamPathTemplate": "/api/{everything}",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        {
          "Host": "localhost",
          "Port": 5002
        }
      ],
      "UpstreamPathTemplate": "/product-api/{everything}"
    },
    {
      "DownstreamPathTemplate": "/api/{everything}",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        {
          "Host": "localhost",
          "Port": 5001
        }
      ],
      "UpstreamPathTemplate": "/order-api/{everything}"
    }
  ],
  "GlobalConfiguration": {
    "BaseUrl": "http://localhost:5000"
  }
}
```

BaseUrl ile gateway'in adresini söylemiş oluyoruz. Routes kısmına mikro servislerimizin adresleri yazıyoruz. 
Örneğin localhost:5000/product-api/..... ile gelen istekler localhost:5002/api/.... olarak yönlendirilecek.

Not: Tüm servisleri ve gateway'i iis üzerinden değil kestrel üzerinden çalıştırıyorum!

Mikro servislerin Startup dosyalarında Map bölümünü aşağıdaki gibi düzenliyorum.
```csharp
app.UseEndpoints(endpoints =>
	{
		endpoints.MapGet("/single", async context =>
		{
			await context.Response.WriteAsync("Hello World! order");
		});

		endpoints.MapGet("/", async context =>
		{
			await context.Response.WriteAsync("Hello World!");
		});
	});
```

Visual Studio Debug ayarlarında Order api nin 5001, Product apinin 5002, GateWay in 5000 üzerinden çalışması için ayarlıyorum.

Solution'a multi debug ile tüm projeleri Kestrel ile çalıştırıyorum ve aşağıdaki adresleri tarayıcıda açıyorum. 

http://localhost:5000/order-api/single
http://localhost:5000/product-api/single

Bu şekilde bir micro servis yapısı çalışır ama çok basit bir yapı. 

Örneğin order api den bir tane daha bu sisteme eklemek istediğimizde, ona ayrı port vermek zorundayız ya da gateway'e manuel olarak hangi servisin nerede olduğunu söylemek pek iyi bişey değil ya da mikro servislerin health check kontrolleri bulunmuyor. İşte bunlar için Consul kullanabiliriz ve GateWay'e servis adresi yerine servis ismi verip tüm bu sorunları consul ile halledebiliriz.