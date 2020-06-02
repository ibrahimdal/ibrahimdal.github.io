---
layout: post
title:  ".Net Core Generic Host Örneği"
description: 
date:   2020-02-09 14:15:36 +0530
categories: .NetCore C#
---
Bu Örnekte .net core 3 ile zamanlanmış (10 saniye de bir) çalışacak basit bir iş parçacığı gösterilmiştir. Kodlara aşağıdaki linkten ulaşabilirsiniz.
https://github.com/ibrahimdal/BlogPostApps/tree/master/SampleForGenericHost


-Program.cs
```javascript
var host = new HostBuilder()
	.ConfigureServices((hostContext, services) =>
	{
		services.AddLogging(); 
		//SampleService classını çalıştırmasını istiyoruz.
		services.AddHostedService<SampleService>();
	})
	.ConfigureLogging((hostContext, configLogging) =>
	{
		configLogging.AddConsole();
		configLogging.AddDebug();
	})
	.UseConsoleLifetime()
	.Build();

await host.RunAsync();
```
-SampleService.cs
```javascript
//SampleService classını  IHostedService den miras aldırıyoruz. 
//Bu sayede clasın bir HostService olduğunu belirtiyoruz. 
//Ayrıca classın içerisinde timer kullandığımız için IDisposable interface'inden miras alıyoruz.
//Bunun sebebi servis durdurulduğunda Timer ın dispose etmek istememiz.
public class SampleService : IHostedService, IDisposable
{
	private readonly ILogger<SampleService> _logger;
	private Timer _timer;

	public SampleService(
		ILogger<SampleService> logger
		)
	{
		_logger = logger;
	}

	//Burada IHostedService in StartAsync fonksiyonunu implemente ediyoruz.
	//Servis ayağa kalktığında bu kısım çalışıyor.
	public Task StartAsync(CancellationToken cancellationToken)
	{
		_logger.LogInformation("Service is starting.");

		//servis ayağa kalktığında 10 saniyede bir çalışacak bir timer oluşturuyor.
		//timer her trigger olduğunda DoWork fonksiyonunu çalıştıracak.
		_timer = new Timer(DoWork, null, TimeSpan.Zero, TimeSpan.FromSeconds(10));

		return Task.CompletedTask;
	}

	//Burada IHostedService in StopAsync fonksiyonunu implemente ediyoruz.
	//Servis durduğunda yapılacak işlemleri içine belirtiyoruz.
	public Task StopAsync(CancellationToken cancellationToken)
	{
		_logger.LogInformation("Service is stopping.");

		//Servis durduğunda timer durdurulacak.
		_timer?.Change(Timeout.Infinite, 0);

		return Task.CompletedTask;
	}

	//DoWork fonksiyonu bizim asıl yapılmasını istediğimiz işi yapıyor.
	private void DoWork(object state)
	{
		_logger.LogInformation("Service is working.");
	}

	//Bu kısımda IDisposable interface inin Dispose fonksiyonunu implemente ediyoruz.
	//Servis dispose olduğunda bu kısımdaki işlemler yapılacak.
	public void Dispose()
	{
		//IDisposeable interfaceinden miras alan timer nesnesini burada dispose ediyoruz.
		_timer?.Dispose();

		//Not: Sizin kullanım şeklinize göre başka dispose olabilen nesneler var ise onlarıda burda dispose etmeniz gerekiyor.
	}
}
```