---
layout: post
title:  ".Net Core Micro Servis ve Servis Discovery Örneği"
description: 
date:   2020-06-03 09:02:36 +0530
categories: .NetCore MicroService ServiceDiscovery Consul Docker
---
Bu yazımda [şuradaki](https://ibrahimdal.github.io/posts/net-core-da-microservice-ornegi/) oluşturduğumuz micro servis yapısını consul ile daha yönetilebilir bir hale getireceğim. Ayrıca tüm projeyi docker üzerinde çalıştıracağım.

Öncelikle ServisDiscovery kavramından bahsedelim. ServisDiscovery -bizim burada kullanacağımız Consul- temel deki görevi hangi servisin hangi adreste olduğunu söylemektir. GateWay 'Ben bir ürün servisi istiyorum' der, ServisDiscovery de ürün servisinin adresini GateWay'e iletir. Ayrıca ServiceDiscovery servislerin ayakta mı? değil mi? -HealthCheck- kontrolünü yapar. Çalışma zamanında yeni bir servis ayağa kalktığında ServiceDiscovery'e 'Ben ürün servisiyim ve bana bu adresten ulaşılabilir, ayrıca beni bu adresten x saniyede bir kontrol edebilirsin.' şeklinde bir bildirimde bulunur, ServiceDiscovery'de bu servisi listesine ekler.

Şu anda projemiz de bir gateway ve 2 adet mikro servis bulunmakta, bu yapıda servis discovery olarak [Consul](https://www.consul.io/) kullanacağız. 

Consul'ü Docker üzerinde kullanacağız. O yüzden önce tüm projeyi Dockerize hale getirelim.

> Önceki projeyi değiştirmeden yeni bir branch açıyorum. 'Step2'

Sırasıyla gateway, productservice, orderservice sağ tıklayıp Add -> Docker Support diyorum ve Linux u seçiyorum. Bu işlemden sonra visual studio projelerimin içine Dockerfile ve .dockerignore dosyalarını kendisi ekliyor. 

```dockerfile
FROM mcr.microsoft.com/dotnet/core/aspnet:3.1-buster-slim AS base
WORKDIR /app
EXPOSE 80

FROM mcr.microsoft.com/dotnet/core/sdk:3.1-buster AS build
WORKDIR /src
COPY ["SampleMicroService.GateWay/SampleMicroService.GateWay.csproj", "SampleMicroService.GateWay/"]
RUN dotnet restore "SampleMicroService.GateWay/SampleMicroService.GateWay.csproj"
COPY . .
WORKDIR "/src/SampleMicroService.GateWay"
RUN dotnet build "SampleMicroService.GateWay.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "SampleMicroService.GateWay.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "SampleMicroService.GateWay.dll"]
```

Örneğin yukarıdaki GateWay'in Dockerfile dosyası, biraz incelersek; 

- aspnet:3.1-buster-slim imajından miras alınmış, çalışma klasörü app olarak belirtilmiş. ve bu imajın 80 portunun dışarıdan erişilebilir olması istenmiş.
- Ardından build işlemine geçilmiş, sdk:3.1-buster imajından miras alınmış, projenin csproj dosyası build stage'ine kopyalanmış, restore denilerek gerekli paketler yüklenmiş. Projenin dosyaları kopyalanmış. /app/build içine proje release olarak build edilmiş.
- Artık bizim elimizde projenin release hali var (buildStage -> /app/build) şimdi bunun publish dosyalarının çıkartılması işlemini yapacağız. Bunun için build stageinden miras alınmış, bu yeni stage'e publish adı verilmiş. build'den miras aldığım için build stageindeki herşey publish'e de geçmiş oluyor. ardından dotnet publish diyerek dosyaların /app/publish içine çıkartıyorum.
- Artık projemin publish dosyaları publish staginde /app/publish içinde mevcut, Docker olamasa ben bu dosyaları .net runtime olan bir bilgisayarda çalıştırmam gerekiyordu. Bu dosyanın base stageinde sadece runtime bulunuyor, o halde base stageinden miras alıyorum. Çalışma klasörünü /app olarak belirtiyorum. Artık publish dosyalarını bu stage'e taşımam gerekiyor, dosyaları publish stage'inden kopyalıyorum. dotnet diyerek projeyi çalıştırıyorum.

Servislerimin hepsini dockerize hale getirdim. Şuanda servislerim docker üzerinden çalışabiliyorlar. Lakin bunların birbirleri ile iletişim kurmaları ve consul ile de iletişimde olmaları gerekiyor, ve benim bu servisleri orkestre etmem gerekiyor (Container Orchestration). Tüm bunlar için docker-compose'u kullanacağım.

GateWay'e sağ tıklıyorum, Add -> Container Orchestration Support diyorum. -birden fazla orkestrasyon seçenekli bulunuyor, biz burda docker compose kullanacağız- docker-compose'u seçiyorum. Linux diyorum. Benim için bir docker-compose.yml dosyası oluşturuyor. Ayrıca docker-compose.override.yml adında bir daha oluşturuyor.

docker-compose.yml 'a imajın ismini ve yolunu söylüyoruz, docker-compose.override.yml dosyasına ise o imajla alakalı konfigurasyon - environment, ports, volume, network - bilgilerini giriyoruz.

Diğer servislere de sağ tıklayarak orchestration support ekleyelim.

```yml
version: '3.4'

services:
  samplemicroservice.gateway:
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
    ports:
      - "80"

  samplemicroservice.orderservice:
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
    ports:
      - "80"


  samplemicroservice.productservice:
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
    ports:
      - "80"
```

yml dosyamız şu anda yukarıdaki gibi, artık consul'u kurabiliriz. docker-compose dosyasına consul'u servisini ekliyorum.

```yaml
services:

  consul:
    image: consul:latest
    command: consul agent -dev -log-level=warn -ui -client=0.0.0.0
    hostname: consul
    container_name: consul
```

Ardından override dosyasına consul'un port mapping işlemini belirtiyorum.

```yaml
services:

  consul:
    ports:
      - "8500:8500"
      - "8600:8600/udp"
```

Böylece consulun kurulumunu tamamladık, http://localhost:8500/ui/dc1/services adresinden consul arayüzüne ulaşıyor olmam gerekiyor.

> Consul olmadan bütün servislere ayrı ayrı 80 81 82 porttan ulaşılacak şekilde ayarladığımızı farz edelim. 80 ile gateway'e, 81 ile productservice'e ulaşıyor oluruz. gateway'e product servisin adresi 81 deyip, gateway üzerinden product servise ulaşmaya çalıştığımızda sonuç elde edemeyiz. Bunun sebebi tüm servislerimiz aynı network te olmamasıdır. Consul olmadan bu işi docker üzerinde yapmak istiyor isek tüm servislerimizi aynı network'e getirmeliyiz. Bunun içinde docker network özelliğini kullanabiliriz. 

Docker-compose dosyalarında biraz değişiklik yapıyorum, gateway'in dışarıya açıyorum ve servislere container_name veriyorum.

```yaml
version: '3.4'

services:

  consul:
    image: consul:latest
    command: consul agent -dev -log-level=warn -ui -client=0.0.0.0
    hostname: consul
    container_name: consul
  
  samplemicroservice.gateway:
    image: ${DOCKER_REGISTRY-}samplemicroservicegateway
    build:
      context: .
      dockerfile: SampleMicroService.GateWay/Dockerfile
    container_name: samplemicroservice.gateway

  samplemicroservice.orderservice:
    image: ${DOCKER_REGISTRY-}samplemicroserviceorderservice
    build:
      context: .
      dockerfile: SampleMicroService.OrderService/Dockerfile
    container_name: samplemicroservice.orderservice


  samplemicroservice.productservice:
    image: ${DOCKER_REGISTRY-}samplemicroserviceproductservice
    build:
      context: .
      dockerfile: SampleMicroService.ProductService/Dockerfile
    container_name: samplemicroservice.productservice
```

Ardında override dosyasını aşağıdaki şekilde son haline ulaştırıyorum. Burada environmentlar girdiğimizi görebiliriz. Bu environment ları servislerin içerisinde kullanacağız.

```yaml
version: '3.4'

services:

  consul:
    ports:
      - "8500:8500"
      - "8600:8600/udp"

  samplemicroservice.gateway:
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
    ports:
      - "80:80"

  samplemicroservice.orderservice:
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ServiceConfig__serviceDiscoveryAddress=http://consul:8500
      - ServiceConfig__serviceName=samplemicroservice.orderservice
      - ServiceConfig__serviceId=1
      - ServiceConfig__serviceAddress=http://samplemicroservice.orderservice:80
      - ServiceConfig__tcpCheckTimeOutAsMin=1
      - ServiceConfig__tcpCheckIntervalAsSec=30
      - ServiceConfig__healtCheckTimeOutAsMin=1
      - ServiceConfig__healtCheckIntervalAsSec=30
    ports:
      - "80"


  samplemicroservice.productservice:
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ServiceConfig__serviceDiscoveryAddress=http://consul:8500
      - ServiceConfig__serviceName=samplemicroservice.productservice
      - ServiceConfig__serviceId=1
      - ServiceConfig__serviceAddress=http://samplemicroservice.productservice:80
      - ServiceConfig__tcpCheckTimeOutAsMin=1
      - ServiceConfig__tcpCheckIntervalAsSec=30
      - ServiceConfig__healtCheckTimeOutAsMin=1
      - ServiceConfig__healtCheckIntervalAsSec=30
    ports:
      - "80"
```

Docker dosyalarında şimdilik işimiz bu kadar. Şimdi servislerimiz ayağa kalktığında consul'e kendilerini bildirmeleri için yapılması gereken değişiklikleri yapalım. Ben bunu sadece orderservis için yapacağım. Productservis için olan işlemler aynı olacaktır.

Burada ihtiyacımız olan mikro servislerin ayağa kalktığında 'ben burdayım' diyerek kendilerini consul'e bildirmeleri, kapandıklarında ise kapandıkları bilgisini consule bildirmeleridir. Bu işlemi miko servislerin içine bir IHostedService örneği inject ederek yapacağız.

SampleMicroService.Infrastructure adında bir class library oluşturuyorum. Bu library içinde micro servislerin yapısal işlemlerini tutacağım.

Nuget'ten aşağıdaki paketleri kuruyorum.

```xml
<PackageReference Include="Consul" Version="0.7.2.6" />
<PackageReference Include="Microsoft.Extensions.Configuration" Version="3.1.4" />
<PackageReference Include="Microsoft.Extensions.Configuration.Binder" Version="3.1.4" />
<PackageReference Include="Microsoft.Extensions.Hosting.Abstractions" Version="3.1.4" />
```

ServiceConfigDto.cs adında bir class oluşturuyorum. Bunun içinde mikro servislerin configuration bilgilerini tutacağız.

```csharp
public class ServiceConfigDto
{
    public Uri ServiceDiscoveryAddress { get; set; }
    public Uri ServiceAddress { get; set; }
    public string ServiceName { get; set; }
    public string ServiceId { get; set; }

    public int TcpCheckTimeOutAsMin { get; set; }
    public int TcpCheckIntervalAsSec { get; set; }
    public int HealtCheckTimeOutAsMin { get; set; }
    public int HealtCheckIntervalAsSec { get; set; }
}
```

ServiceDiscoveryHostedService.cs adında bir class oluşturuyorum ve IHostedService interface'ini implemente ediyorum.

```csharp
public class ServiceDiscoveryHostedService : IHostedService
    {
        private readonly IConsulClient _client;
        private readonly ServiceConfigDto _config;
        private string _registrationId;

        public ServiceDiscoveryHostedService(
            IConsulClient client,
            ServiceConfigDto serviceConfig
            )
        {
            _client = client;
            _config = serviceConfig;
        }

    	///bu kısım servis ayağa kalktığında çalışır.
        public async Task StartAsync(CancellationToken cancellationToken)
        {
            ///her servis için uniq bir registrationId oluşturuyoruz.
            _registrationId = $"{_config.ServiceName}-{_config.ServiceId}";
		///consule gönderilecek registration bilgisini oluşturuyoruz.
            var registration = new AgentServiceRegistration
            {
                ID = _registrationId, //servis reg Id
                Name = _config.ServiceName, //servis adı
                Address = _config.ServiceAddress.Host, //servis adresi
                Port = _config.ServiceAddress.Port, //servis portu
                Checks = new AgentServiceCheck[] //servisin nasıl kontrol edilmesini istersek.
                {
                    //servisi Interval aralıkla kontrol et. DeregisterCriticalServiceAfter kadar süre içinde yanıt gelmez ise servis kapalı kabul et. TCP üzerinden kontrol et.
                    new AgentServiceCheck
                    {
                        DeregisterCriticalServiceAfter = TimeSpan.FromMinutes(_config.TcpCheckTimeOutAsMin),
                        Interval = TimeSpan.FromSeconds(_config.TcpCheckIntervalAsSec),
                        TCP = $"{_config.ServiceAddress.Host}:{_config.ServiceAddress.Port}"
                    },
                    //servisi Interval aralıkla kontrol et. DeregisterCriticalServiceAfter kadar süre içinde yanıt gelmez ise servis kapalı kabul et. HTTP üzerinden kontrol et.
                    new AgentServiceCheck
                    {
                        DeregisterCriticalServiceAfter = TimeSpan.FromMinutes(_config.HealtCheckTimeOutAsMin),
                        Interval = TimeSpan.FromSeconds(_config.HealtCheckIntervalAsSec),
                        HTTP = $"{_config.ServiceAddress.Scheme}://{_config.ServiceAddress.Host}:{_config.ServiceAddress.Port}/HealthCheck"
                    }
                }
            };
			///daha önce bu reg id ile kayıtlı servisin kaydını varsa sil.
            await _client.Agent.ServiceDeregister(registration.ID, cancellationToken);
            ///yeni servisi kaydet.
            await _client.Agent.ServiceRegister(registration, cancellationToken);
        }
		///bu kısım servis durdurulduğunda çalışır.
        public async Task StopAsync(CancellationToken cancellationToken)
        {
            ///consule servis kapandı bilgisini iletir.
            await _client.Agent.ServiceDeregister(_registrationId, cancellationToken);
        }
    }
```

Ardından infrastructure kütüphanesini mikro servislerde kullanmak için StartUpExtentions adında bir static class oluşturuyorum. GetServiceConfig fonksiyonu configuration içindeki değerleri ServiceConfigDto tipinde bir nesneye dönüştürüyor.

RegisterConsulServices fonksiyonu ise consul'e istek göndermemiz için bir client oluşturuyor. Consul client'ını servis builder'a inject ediyor. Ayrıca ServiceDiscoveryHostedService classından, servis ayağa kalktığında bir adet oluşmasını sağlıyor. Biz bu fonksiyonları mikro servislerin startup larında kullanacağız.

```csharp
using Consul;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using System;

namespace SampleMicroService.Infrastructure
{
    public static class StartUpExtentions
    {
        public static ServiceConfigDto GetServiceConfig(
            this IConfiguration configuration
            )
        {
            return new ServiceConfigDto
            {
                ServiceDiscoveryAddress = configuration.GetValue<Uri>("ServiceConfig:serviceDiscoveryAddress"),
                ServiceAddress = configuration.GetValue<Uri>("ServiceConfig:serviceAddress"),
                ServiceName = configuration.GetValue<string>("ServiceConfig:serviceName"),
                ServiceId = configuration.GetValue<string>("ServiceConfig:serviceId"),
                TcpCheckTimeOutAsMin = configuration.GetValue<int>("ServiceConfig:tcpCheckTimeOutAsMin"),
                TcpCheckIntervalAsSec = configuration.GetValue<int>("ServiceConfig:tcpCheckIntervalAsSec"),
                HealtCheckTimeOutAsMin = configuration.GetValue<int>("ServiceConfig:healtCheckTimeOutAsMin"),
                HealtCheckIntervalAsSec = configuration.GetValue<int>("ServiceConfig:healtCheckIntervalAsSec")
            };
        }

        public static void RegisterConsulServices(
            this IServiceCollection services,
            ServiceConfigDto serviceConfig
            )
        {
            var consulClient = new ConsulClient(config =>
            {
                config.Address = serviceConfig.ServiceDiscoveryAddress;
            }
             );

            services.AddSingleton(serviceConfig);
            services.AddSingleton<IConsulClient, ConsulClient>(p => consulClient);
            services.AddSingleton<IHostedService, ServiceDiscoveryHostedService>();
        }
    }
}
```

Infrastructure hazır, aşağıdaki gibi mikro servislere olarak ekleyelim.

```xml
<ProjectReference Include="..\..\SampleMicroService.Infrastructure\SampleMicroService.Infrastructure.csproj" />
```

Önceki yazımızda mikro servislere controller eklememiştik. Şimdi ekleyelim.

DemoController oluşturuyorum.

```csharp
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;
using System.Threading.Tasks;

namespace SampleMicroService.OrderService.Controllers
{
    [Route("api/[controller]")]
    [ApiController]
    public class DemoController : ControllerBase
    {
        private readonly IHttpContextAccessor _httpContextAccessor;
        public DemoController(
            IHttpContextAccessor httpContextAccessor
            )
        {
            _httpContextAccessor = httpContextAccessor;
        }

        /// <summary>
        /// Demo değerleri döner
        /// </summary>
        /// <returns>Dizi döner</returns>
        [HttpGet]
        [Route("List")]
        public async Task<IActionResult> GetList()
        {
            return Ok(new string[] { "item1", "item2", "item3" });
        }

        /// <summary>
        /// Tek bir demo değeri döner
        /// </summary>
        /// <param name="id">demo değer id</param>
        /// <returns>demo değeri</returns>
        [HttpGet]
        [Route("Single/{id}")]
        public async Task<IActionResult> GetSingle(int id)
        {
            return Ok("item" + id);
        }
    }
}
```

Dikkat ettiyseniz Consul e /HealthCheck adresinden bizi kontrol edebileceğini söylemiştik. Onun için bir HealthCheckController oluşturuyorum.

```csharp
using Microsoft.AspNetCore.Mvc;

namespace SampleMicroService.OrderService.Controllers
{
    [ApiController]
    [Route("[controller]")]
    public class HealthCheckController : ControllerBase
    {
        [HttpGet("")]
        [HttpHead("")]
        public IActionResult Ping()
        {
            return Ok();
        }
    }
}
```

Şimdi Mikro servisin startup'ında infrastrucure'ın fonksiyonlarını kullanalım.

```csharp
public class Startup
    {
        /// <summary>
        /// Configuration'ı inject alıyoruz.
        /// </summary>
        public IConfiguration Configuration { get; }
        public Startup(IConfiguration configuration)
        {
            Configuration = configuration;
        }

        public void ConfigureServices(IServiceCollection services)
        {
            //Configuration içindeki değerleri alıyoruz.
            var serviceConfig = Configuration.GetServiceConfig();
            //Servisin consul'e kayıt olması için ayarlamaları uyguluyoruz.
            services.RegisterConsulServices(serviceConfig);

            services.AddControllers();
            services.AddHttpContextAccessor();
        }

        // This method gets called by the runtime. Use this method to configure the HTTP request pipeline.
        public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
        {
            if (env.IsDevelopment())
            {
                app.UseDeveloperExceptionPage();
            }

            app.UseRouting();

            app.UseEndpoints(endpoints =>
            {
                endpoints.MapControllers();
                endpoints.MapGet("/version", async context =>
                {
                    await context.Response.WriteAsync($"v{Assembly.GetExecutingAssembly().GetName().Version}");
                });
            });
        }
    }
```

Projeye docker-compose up diyelim ve localhost:8500 adresine girelim. OrderService ve ProductService in başarılı bir şekilde kayıtlı olduğunu görebiliriz. Ayrıca Service Check menüsünden kayıt olurken gönderdikleri HealtCheck ayarlarını görebiliriz.

Mikro servislerimizi ve servis discovery'i ayağa kaldırdık. Geriye şimdi sıra gateway'in service discovery ile haberleşmesinde. Bunun için ocelot.json dosyasında aşağıdaki gibi değişiklik yapalım.

```json
{
  "Routes": [
    {
      "DownstreamPathTemplate": "/api/{everything}",
      "DownstreamScheme": "http",
      "ServiceName": "samplemicroservice.orderservice",
      "UpstreamPathTemplate": "/order-api/{everything}"
    },
    {
      "DownstreamPathTemplate": "/api/{everything}",
      "DownstreamScheme": "http",
      "ServiceName": "samplemicroservice.productservice",
      "UpstreamPathTemplate": "/product-api/{everything}"
    }
  ],
  "GlobalConfiguration": {
    "ServiceDiscoveryProvider": {
      "Host": "consul",
      "Port": 8500,
      "Type": "Consul"
    }
  }
}
```

GlobalConfiguration kısmında bir serviceDiscoveryProvider tanımlıyoruz, bunun tipi, host ve portunu bildiriyoruz. Routes kısmında önceki gibi mikro servislerimize olan yönlendirme ayarlarını giriyoruz. Dikkatinizi çekmek istediğim konu; eskiden servisin adresini giriyor iken şimdi sadece servisin ismini 'ServiceName' giriyoruz. Bu servis ismi servislerin consul'e kayıt olduğunda kullandığı isim ile aynı olmalıdır.

Ardında GateWay'in program.cs dosyasında da değişiklikler yapmalıyız. ocelot'a servis provider olarak consul'u nasıl kullanacağını söylememiz gerekiyor. Bunun için aşağıdaki paketi GateWay'e kuralım.

```xml
<PackageReference Include="Ocelot.Provider.Consul" Version="16.0.1" />
```

Program.cs dosyasında ise aşağıdaki değişiklikleri yapalım.

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using Ocelot.DependencyInjection;
using Ocelot.Middleware;
using Ocelot.Provider.Consul;
using System.IO;

namespace SampleMicroService.GateWay
{
    public class Program
    {
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
                    //AddConsul.
                    services.AddOcelot().AddConsul();
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
    }
}
```

Artık tüm yapımız hazır durumda. Aşağıdaki adreslere girerek kontrol edebiliriz.

http://localhost/product-api/demo/single/1

http://localhost/product-api/demo/List

http://localhost/order-api/demo/List

Artık bundan sonra order ya da product servisin birden fazla çalıştırılması ve gateway tarafından load balance edilme işlemleri size kalmış.  