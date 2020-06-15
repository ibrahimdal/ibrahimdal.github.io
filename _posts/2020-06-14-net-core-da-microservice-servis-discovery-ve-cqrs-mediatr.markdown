---
layout: post
title:  ".Net Core Micro Servis ve CQRS - MediatR"
description: 
date:   2020-06-14 22:01:36 +0530
categories: .NetCore MicroService Swagger
---
Son oluşturduğumuz microservice yapımıza mediatR implementasyonu yapacağız. MediatR ve CQRS ile ilgili daha detaylı bilgileri başka sayfalarda bulabilirsiniz, o yüzden bu konuya fazla değinmeyeceğim.

Kısaca cqrs si açıklamak gerekirse; örneğin kullanıcı ekleme ve kullanıcı gösterme ve kullanıcı listeleme işlemleri yapıyoruz, genelde bunun için UserDto ve UserEnttiy oluşturmak ve tüm bu işlemlerde UserDto clasını kullanmak, ama burada bir işlemde UserDto içindeki bir yada birkaç field'ı kullanırken diğerinde kullanmıyoruz, bunun önüne geçmek içinde cqrs yaklaşımından faydalanıp her işlem için ayrı class (command yada query) tanımlıyoruz ve bu classların handler larında gerekli işlemleri gerçekleştiriyoruz. Yani tüm işlemleri birbirinde bir nevi soyutluyoruz, bunun bize tamamen proje geliştirirken faydası oluyor.

Burada bilinmesi gereken command ve query kavramları; command: uygulamanın durumunu değiştiriken, query: uygulamanın var olan durumunu göstermeye yarıyor. command ile insert,update,delete işlemleri yaparken, query ile sadece read işlemini gerçekleştiriyoruz.

MediatR ise CQRS yaklaşımını uygulamamıza yarayan bir paket, bu paketi kullanmadan da CQRS yaklaşımı uygulanabilir.

src -> core  altına SampleMicroService.Application adında bir library açıyorum ve ilk olarak MediatR.Extensions.Microsoft.DependencyInjection paketini nuget'ten yüklüyorum.

Genel olarak uygulamamın tüm kodları SampleMicroService.Application içinde olacak, SampleMicroService.Application'ı tüm api servislerime refere edeceğim ve startuplarında Application içinde kullanılan servisleri inject edeceğim.

İlk inject edeceğim servis tabiki MediatR'ın kendisi, tüm startuplarda ayrı ayrı kod yazmak yerine Application içine bir DependencyInjection oluşturuyorum.

```c#
public static class DependencyInjection
    {
        public static IServiceCollection AddApplication(this IServiceCollection services)
        {
            services.AddMediatR(Assembly.GetExecutingAssembly());

            return services;
        }
    }
```

Şimdi bunu OrderService'ine ekleyelim. StartUp->configureServices içine aşağıdaki gibi;

```c#
 services.AddApplication();
```

OrderService içine aşağıdaki gibi bir ApiController ekliyoruz. Bundan sonra tüm controller'larımızı bundan türeteceğiz.

```c#
using MediatR;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.DependencyInjection;

namespace SampleMicroService.OrderService.Controllers
{
    [ApiController]
    [Route("api/[controller]")]
    public abstract class ApiController : ControllerBase
    {
        private IMediator _mediator;

        protected IMediator Mediator => _mediator ??= HttpContext.RequestServices.GetService<IMediator>();
    }
}
```

Ardından OrderController oluşturuyoruz ve içine bir tane Get metodu ekliyoruz.

```c#
using Microsoft.AspNetCore.Mvc;
using System.Threading.Tasks;

namespace SampleMicroService.OrderService.Controllers
{
    public class OrderController : ApiController
    {
        /// <summary>
        /// id ye göre order bilgisi döndürür.
        /// </summary>
        /// <param name="id"></param>
        /// <returns></returns>
        [HttpGet("{id}")]
        public async Task<ActionResult<OrderDto>> Get(int id)
        {
            return await Mediator.Send(new GetOrderQuery { orderId = id });
        }
    }
}
```

Yukarda olan olay kısaca şu bir id geliyor ve bu id yi içinde barındıran bir GetOrderQuery tipinde bir nesne oluşuyor ve bu nesne mediatr'a gönderiliyor dönen sonuç ise OrderDto tipinden olması isteniyor.

Bunun için ilk olarak Application içine Order adında bir klasör oluşturalım. Ardından Order klasörü içine de Commands ve Queries adında iki klasör oluşturalım. Order ile alakalı tüm işlemler Order klasörü içinde yapılacak.

Örnek olarak bu işlem bir order'ın bilgisini geri dönderdiği için bir Query olması gerekiyor. Bunun için Order -> Queries içine GetOrder Klasörü açalım ve GetOrderResult clasımızı oluşturalım.

```c#
using System;

namespace SampleMicroService.Application.Order.Queries.GetOrder
{
    public class GetOrderResult
    {
        public int id { get; set; }
        public string code { get; set; }
        public decimal totalAmount { get; set; }
        public DateTime createdDate { get; set; }
    }
}

```

Daha sonra GetOrderQuery adında bir class daha oluşturalım.

```c#
using MediatR;

namespace SampleMicroService.Application.Order.Queries.GetOrder
{
    public class GetOrderQuery : IRequest<GetOrderResult>
    {
        public int orderId { get; set; }
    }
}
```

GetOrderQuery classı bizim istek classımız, IRequest den miras alan ve döndürceği sonucu IRequest'e generic olarak bildiren bir class. MediatR kendisine GetOrderQuery tipinde bir istek geldiğinde o isteği handle eder ama bizim o handle işleminin nasıl yapılacağını MediatR'a söylememiz gerekiyor. (Bu anlatılan Command tipleri için de aynı şekildedir.) Bunun için bir handle classı oluşuruyoruz.

```c#
public class GetOrderQueryHandler : IRequestHandler<GetOrderQuery, GetOrderResult>
    {
        public async Task<GetOrderResult> Handle(
            GetOrderQuery request,
            CancellationToken cancellationToken
            )
        {
            return new GetOrderResult
            {
                code = "ORDERCODE",
                createdDate = DateTime.UtcNow,
                id = request.orderId,
                totalAmount = 19.99M
            };
        }
    }
```

Projeyi çalıştırıp http://localhost/order-api/order/2 adresine girerek sonucu görebiliriz.

Birde Commad'a örnek olması açısından order controller'a MakeOrder adında bir metod ekleyelim.

```
/// <summary>
/// Yeni bir sipariş oluşturur.
/// </summary>
/// <param name="command"></param>
/// <returns></returns>
[HttpPost]
public async Task<ActionResult<int>> Create(CreateOrderCommand command)
{
	return await Mediator.Send(command);
}
```

Şimdi CreateOrderCommand class'ını ve Handler'ını oluşturalım.

```c#
using MediatR;
using SampleMicroService.Domain;
using System;
using System.Threading;
using System.Threading.Tasks;

namespace SampleMicroService.Application.Order.Commands.CreateOrder
{
    public class CreateOrderCommand : IRequest<int>
    {
        public decimal totalAmount { get; set; }
        public string orderCode { get; set; }
    }

    public class CreateOrderCommandHandler : IRequestHandler<CreateOrderCommand, int>
    {
        public async Task<int> Handle(
            CreateOrderCommand request, 
            CancellationToken cancellationToken
            )
        {
            var newOrder = new OrderDto
            {
                id = 1,
                createdDate = DateTime.UtcNow,
                orderCode = request.orderCode,
                totalAmount = request.totalAmount
            };

            //save.....to anywhere

            return newOrder.id;
        }
    }
}

```

Yukarda bir SampleMicroService.Domain library'si oluşturduk ve içine OrderDto classını koyduk. Bu, uygulama içinde order işlemlerini gerçekleştirirken kullanacağımız, order'ın yapısını tutan bir class. Daha sonra bu classı entity'e map'leyip veritabanına yazacağız. 

MediatR temel olarak bu şekilde, birazda MediatR'ın pipeline özelliğine bakalım.

Pipeline kısaca veri bir yerden bir yere akarken araya girip bazı işlemler yapmamıza olanak tanıyan sistemlerdir.

MediatR daki pipeline lar ile örneğin loglama, performance kontrolü, exception yakalama, validation işlemlerini yapabiliriz. Hatta authentication kontrolünü de yapabiliriz.

Önce Application'a Common adında bir klasör ekleyelim. Common içine de Behaviours adında bir klasör ekleyelim.

İlk olarak performans kontrolünü yapacak bir pipeline oluşturalım. Kısaca çalışması x ms'den fazla süren bir istek olduğunda bize log atacak.

PerformanceBehaviour.cs adında bir dosya oluşturuyorum.

```c#
using MediatR;
using Microsoft.Extensions.Logging;
using System.Diagnostics;
using System.Threading;
using System.Threading.Tasks;

namespace SampleMicroService.Application.Common.Behaviours
{
    public class PerformanceBehaviour<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
    {
        private readonly Stopwatch _timer;
        private readonly ILogger<TRequest> _logger;

        public PerformanceBehaviour(
            ILogger<TRequest> logger
            )
        {
            _timer = new Stopwatch();

            _logger = logger;
        }
        public async Task<TResponse> Handle(
            TRequest request,
            CancellationToken cancellationToken,
            RequestHandlerDelegate<TResponse> next
            )
        {
            _timer.Start();

            var response = await next(); //istek işleniyor.

            _timer.Stop();

            //500 ms den fazla sürmüş ise; logla.
            var elapsedMilliseconds = _timer.ElapsedMilliseconds;
            if (elapsedMilliseconds > 500)
            {
                var requestName = typeof(TRequest).Name;

                _logger.LogWarning(
                    "SampleMicroService Long Running Request: {Name} ({ElapsedMilliseconds} milliseconds) {@Request}",
                    requestName,
                    elapsedMilliseconds,
                    request
                    );
            }

            return response;
        }
    }
}

```

Şimdi bu pipeline'ı MediatR'a kaydetmemiz gerekiyor. DependecyInjection.cs ye gidiyoruz ve aşağıdaki satırı ekliyoruz.

```c#
services.AddTransient(typeof(IPipelineBehavior<,>), typeof(PerformanceBehaviour<,>));
```

Aslında bu pipeline lar .net core api lardaki middleware yada ActionFilter'a benzetilebilir. Ama biz burada pipeline'ı sanki bir servismiş gibi .net core uygulamamızın serviscollection kısmına ekledik. Bunun sebebi MediatR servis collection'daki IPipelineBehavior tipinde olan servisleri tarayıp onları çalıştırmasıdır.

Ayrıca ILogger servisinide loglama için kullandık burda,ama hiç bir configuration yapmadan, ilerleyen bölümlerde loglama işlemlerine de değineceğiz. Bir noktada şu ki, Command ve Query lerdede ILogger gibi serviceCollection'a kaydedilen servisler kullanılabilir. İlerleyen bölümlerde veritabanına ulaşmaya yarayan servisleri bu şekilde inject ederek kullanacağım.

Şimdi exception ları loglamak için bir pipeline oluşturalım. UnhandledExceptionBehaviour.cs olsun ismi.

```c#
using MediatR;
using Microsoft.Extensions.Logging;
using System;
using System.Threading;
using System.Threading.Tasks;

namespace SampleMicroService.Application.Common.Behaviours
{
    public class UnhandledExceptionBehaviour<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
    {
        private readonly ILogger<TRequest> _logger;

        public UnhandledExceptionBehaviour(ILogger<TRequest> logger)
        {
            _logger = logger;
        }

        public async Task<TResponse> Handle(TRequest request, CancellationToken cancellationToken, RequestHandlerDelegate<TResponse> next)
        {
            try
            {
                return await next();
            }
            catch (Exception ex)
            {
                var requestName = typeof(TRequest).Name;

                _logger.LogError(ex, "SampleMicroService Request: Unhandled Exception for Request {Name} {@Request}", requestName, request);

                throw;
            }
        }
    }
}

```

Şimdi de Logging adında bir pipeline oluşturalım. 

```c#
using MediatR.Pipeline;
using Microsoft.Extensions.Logging;
using System.Threading;
using System.Threading.Tasks;

namespace SampleMicroService.Application.Common.Behaviours
{
    public class LoggingBehaviour<TRequest> : IRequestPreProcessor<TRequest>
    {
        private readonly ILogger _logger;

        public LoggingBehaviour(
            ILogger<TRequest> logger
            )
        {
            _logger = logger;
        }

        public async Task Process(TRequest request, CancellationToken cancellationToken)
        {
            //kim hangi isteği yapmış logla.
        }
    }
}
```

Dikkat ettiyseniz bu pipeline aldığı isteği bir yere next lemiyor ve IRequestPreProcessor interface'ini implemente ediyor. Kısaca şu işi yapıyor; istek geldiğinde isteği dinliyor ve isteğin bir kopyasını alıyor. İstek burada yoluna devam ediyor. Bu pipeline ise isteğin normal çalışmasına paralel çalışıyor. Biz bu pipeline'ı kim hangi isteği yapmış bilgisini loglamak için kullanıyoruz. Ayrıca IRequestPreProcessor tipindeki pipeline ları serviceCollection'a kaydetmemize gerek yok, MediatR bunları kendisi çalıştırıyor.

Bir diğer pipeline'ımız da Validation adında olacak. İstekler işlenmeden önce validation işlemini gerçekleştirecek ama şimdilik implemente etmiyoruz, içini boş bırakıyoruz.

```c#
using MediatR;
using System.Threading;
using System.Threading.Tasks;

namespace SampleMicroService.Application.Common.Behaviours
{
    public class ValidationBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
        where TRequest : IRequest<TResponse>
    {
        public async Task<TResponse> Handle(TRequest request, CancellationToken cancellationToken, RequestHandlerDelegate<TResponse> next)
        {
            return await next();
        }
    }
}
```

Son olarak DepencyInjection classının son halide bu şekilde.

```c#
using MediatR;
using Microsoft.Extensions.DependencyInjection;
using SampleMicroService.Application.Common.Behaviours;
using System.Reflection;

namespace SampleMicroService.Application
{
    public static class DependencyInjection
    {
        public static IServiceCollection AddApplication(this IServiceCollection services)
        {
            services.AddMediatR(Assembly.GetExecutingAssembly());
            services.AddTransient(typeof(IPipelineBehavior<,>), typeof(PerformanceBehaviour<,>));
            services.AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
            services.AddTransient(typeof(IPipelineBehavior<,>), typeof(UnhandledExceptionBehaviour<,>));
            return services;
        }
    }
}

```

Burada pipelineların sırası önemli. 