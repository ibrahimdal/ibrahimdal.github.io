---
layout: post
title:  "Angular Projesini Dockerize Etme"
description: 
date:   2020-02-17 14:15:36 +0530
categories: Angular Docker
---
Bu Örnekte default bir angular projesini development ve prod ortamlarda nasıl dockerize edileceği anlatılmıştır. Proje Reposu: https://github.com/ibrahimdal/BlogPostApps

Bilgisayarımıza angular cli kuruyoruz. (-g ibaresi general olarak kurmamızı sağlıyor.)
```shell
npm install -g @angular/cli
```
Dockerize etme işlemi için elimizde bir angular projesi olması gerekiyor. Aşağıdaki komut ile yeni bir angular projesi oluşturalım.
```shell
ng new sampleangular
```
Proje oluşturuldu, şimdi projemizi Visual Studio Code ile açalım. 

Projemiz hazır ng serve diyerek projemizin çalışıp çalışmadığını görebiliriz. Projemiz çalıştığına göre artık yavaş yavaş dockerize etme işlemlerine girelim.

Docker hakkında internette tonla kaynak olduğu için bahsetmeyeceğim, sadece Dockerfile docker ı komut satırında kullanırken yazdığımız komutları kayıtlı tutmak ve docker image'ı oluşturmak için kullandığımız bir dosya, aksini belirtmediğimiz sürece docker çalıştığı dizinde "Dockerfile" ismindeki bu dosyayı arar ve çalıştırır.

Projemizi dockerize etmek için öncelikle bir imaj haline getirmemiz gerekiyor, bunun içinde proje ana dizinine Dockerfile isminde bir dosya oluşturuyorum.

*Dockerfile
```shell
#her docker imajı bir başka imajdan miras almak zorundadır.
#bizde burada npm in gerekliliği olan node'un dockerHub daki imajından miras aldık.
FROM node:12.16.0

#spa projelerinde testleri çalıştırmamız için chrome'a ihtiyacımız var
#node imajı aslında linux imajından miras almışdurumda onun için linux koşturduğumuz komutları burada da koşturabiliriz.
#docker daki RUN komutu shell komutlarını çalıştırmamızı sağlıyor.
RUN wget -q -O - https://dl-ssl.google.com/linux/linux_signing_key.pub | apt-key add -
RUN sh -c 'echo "deb [arch=amd64] http://dl.google.com/linux/chrome/deb/ stable main" >> /etc/apt/sources.list.d/google.list'
RUN apt-get update && apt-get install -yq google-chrome-stable

#çalışma klasörünün /app olarak belirtiyoruz.
#bu komut bizim için bu imaj çalıştığında oluşacak container içinde /app adında bir klasör oluşturacak.
WORKDIR /app

#burada projemiz içinde node_modules'ü container içindeki /app klasörüne mapliyoruz.
ENV PATH /app/node_modules/.bin:$PATH

#projemizdeki package.json dosyasını container içine kopyalıyoruz.
COPY package.json /app/package.json

#package.json içerisindeki paketlerin container'a yüklüyoruz.
RUN npm install

#container'a angular/cli ı general olarak kuruyoruz.
RUN npm install -g @angular/cli@9.0.2 --unsafe

#proje dosyalarını container içine kopyalıyoruz. --.dockerignore dakiler hariç
COPY . /app

#container içerisinde projemizi serve ediyoruz.
CMD ng serve --host 0.0.0.0
```
Bir projeyi dockerize ettiğimizde bazı dosyaların docker imajı içersine alınmamasını isteyebiliriz. Hangi dosyalar olacağı projeden projeye ve kullanılan teknolojiye göre değişebilir. Şu şekilde düşünürsek; biz bu projeyi bir yere taşıdığımıza gerek olmayan dosyalar hangileri ise docker imajına da onları taşımamıza gerek yok. Bu dosyaları docker cli a bildiriken .dockerignore dosyasını kullanıyoruz.

*.dockerignore
```shell
node_modules
.git
.gitignore
```
Docker imajımızı artık oluşturabiliriz, bu imajı development ortamında kullanacağız. Aşağıdaki komutu Dockerfile ın bulunduğu dizinde çalıştırmamız yeterli. -t diyerek imaj'a bir tag atıyoruz. Dockerfile ile aynı dizinde olduğumuz için . diyerek bu dizindeki Dockerfile'ı build et demiş oluyoruz.
```shell
docker build -t sampleangular:dev .
```
Sıra docker imajından konteynır yaratmaya geldi. Bunun için aşağıdaki komutu kullanabiliriz.
```shell
docker run --rm sampleangular:dev
```
Daha sonra localhost:4200 a girip projemize bakalım, herhangi bir sonuç almadığımızı görüyoruz. Bunun sebebi biz imajı çalıştırdık ama bizim projemiz konteynır içindeki 4200 portundan yayınlanır durumda bizim bu porta kendi host bilgisayarımızdan ulaşmamış için port mapleme yapmamız gerekiyor.

Aşağıdaki komut ile docker imajını tekrar çalıştıralım. burada host bilgisayarındaki 4201 portunu konteynır içindeki 4200 a maplemiş olduk. http://localhost:4201 adresine girerek projenin çalıştığını görebiliriz.
```shell
docker run -p 4201:4200 --rm sampleangular:dev
```
Proje çalışırken AppComponent te bir değişiklik yapalım ve projenin bundan etkilenip etkilenmediğine bakalım. Projenin etkilenmediğini görebiliriz. Fakat biz bundan sonra projemizi geliştirme sürecine gireceğiz ve hotreload ile docker konteynır içindeki projeye yansımasını istiyoruz bunun için volume işlemini kullanacağız. Yukarıdaki imajı çalıştırdığımız kodu aşağıdaki hale getiriyoruz.
```shell
docker run -v ${PWD}:/app -v ${PWD}/node_modules:/app/node_modules -p 4201:4200 --rm sampleangular:dev
```
Burada konteynırdaki /app içindeki proje dosyalarını ve /node_module host bilgisayarımızdaki dosyalarımız üzerinden çalışması gerektiğini söylüyoruz. Bundan sonra AppComponent te değişiklik yaptığımızda yada projeye yeni bir npm paketi eklediğimizde konteynır içinde çalışan projemiz bundan etkilenecektir.

Testlerimizi çalıştırmamız için aşağıdaki değişikleri yapalım.
- src/karma.conf.js
```json
browsers: ['ChromeHeadless'],
customLaunchers: {
	'ChromeHeadless': {
		base: 'Chrome',
		flags: [
			'--no-sandbox',
			'--headless',
			'--disable-gpu',
			'--remote-debugging-port=9222'
		]
	}
},
```
- e2e/protractor.conf.js
```json
capabilities: {
	'browserName': 'chrome',
	// new
	'chromeOptions': {
	'args': [
		'--no-sandbox',
		'--headless',
		'--window-size=1024,768'
	]
}
},
```
Konteynır çalışır durumda iken aşağıdaki komut ile konteynır içine komut gönderip testleri çalıştırabiliriz.
```shell
docker exec -it CONTAINER_NAME ng test --watch=false
```
konteynır çalışır durumda iken aşağıdaki komut ile konteynır içine komut gönderip e2e testleri çalıştırabiliriz.
```shell
docker exec -it CONTAINER_NAME ng e2e --port 4202
```
Aşağıdaki komut ile konteynırı durdurabiliriz.
```shell
docker stop CONTAINER_NAME
```
Buraya kadar herşey güzeldi ama projemizi dockerize ettik ve geliştirme ortamında docker'ı kullandık. Ama konteynırı çalıştırıken port ve volume mapleme işlemleri gerçekleştirdik, ayrıca yukarda yapmadık ama konteynır oluştururken sabit bir isim de girmemiz yararlı olacaktır. Farklı bir durum belki ama burada biz tek proje çalıştırdık, bu proje çalışırken başka bir servis (mock service) vs ye ihtiyaç duyabilirdi. İşte bunları her zaman yapmak zaman kaybı ve hatalara sebep olacağından docker-compose ile bunları kayıt altına almak güzel olacaktır.

Bunun için ana dizine docker-compose.yml dosyası oluşturuyoruz. (Başka tipte teknolojiler kullanırken bu dosyayı ana dizinin bir üstüne oluşturmak daha güzel olacaktır. Çünkü birden fazla projemizi aynı anda tek bir docker-compose dosyasıyla ayağa kaldırmak isteyebiliriz. Ama bu proje için gerekmiyor.

*docker-compose.yml
```markdown
version: '3.7'
	services:
		sampleangular2:
			container_name: sampleangular2
			build:
				context: .
				dockerfile: Dockerfile
			volumes:
				- '.:/app'
				- '/node_modules:/app/node_modules'
			ports:
				- '4201:4200'
```
Yukarıdaki docker-compose dosyasını biraz açıklayalım. versionunu 3.7 olarak belirttik. Tüm docker-compose dosyalarında version yazmalısınız.

Ardında servislerimizin olduğunu belirttik. Servisimizin isminin sampleangular2 olduğunu ve bu serviste çalışacak konteynırın isminin sampleangular2 olduğunu, servisin build edilirken kullanacağı Dockerfile ı ve onun path'ini belirttik. Servis çalışırken olması gereken volume işlemlerini belirttik ve port mapleme işlemini gerçekleştirdik. Bunlardan başka docker-compose komutlarına aşağıdaki adresten ulaşabilirsiniz.
https://docs.docker.com/compose/compose-file/

Şimdi aşağıdaki komut ile docker-compose dosyasını çalıştıralım.
```shell
docker-compose up -d --build
```
Aşağıdaki komutlar ile testlerimizi çalıştırabiliriz.
```shell
docker-compose exec sampleangular2 ng test --watch=false
docker-compose exec sampleangular2 ng e2e --port 4202
```
Ardından docker-compose ile çalıştırdığımız servisi durduralım.
```shell
docker-compose stop
```
Bu aşamaya kadar development ortamını tamamlamış bulunmaktayız. Projemizi node,angular cli kurulu olamayan bir ortama taşıdığımızda bile çalışacaktır bunun için sadece docker kurulu olması yeterlidir.

Ama biz production ortama da dockerize edilmiş hali ile çıkmak istiyoruz. Prod ortama çıkarken yukardaki Dockerfile dosyasını kullanabiliriz, ama bunun içinde node, angular/cli, chrome gibi production ortamda gerek olmayan programlar var.

Bir düşünelim eğer docker kullanmadan bu projeyi yayınlasaydık bize ne gerekecekti? ng build ile projenin publish dosyalarını çıkaracaktık ve nginx üzerinden bu dosyaları yayınlayacaktık.

*Dockerfile-prod
```shell
#Burada önce projeyibuild ediyoruz, ardından çıkan dosyaları nginx üzerinden yayınlıyoruz.
#Dev ortamındaki gibi base image olarak node u aldık.
FROM node:12.2.0 as build

#Dev ortamındaki gibi testleri çalıştırmak için chrome yüklüyoruz.
RUN wget -q -O - https://dl-ssl.google.com/linux/linux_signing_key.pub | apt-key add -
RUN sh -c 'echo "deb [arch=amd64] http://dl.google.com/linux/chrome/deb/ stable main" >> /etc/apt/sources.list.d/google.list'
RUN apt-get update && apt-get install -yq google-chrome-stable

#Dev ortamındaki gibi /app klasörünü çalışma dizini olarak belirledik.
WORKDIR /app
#Dev ortamındaki gibi /app/node_modules/.bin container ın path ine yazdık.
ENV PATH /app/node_modules/.bin:$PATH

#Dev ortamındaki gibi projenin package dosyasını container a taşıdık ve bağımlılıkları yükledik.
COPY package.json /app/package.json

RUN npm install
RUN npm install -g @angular/cli@7.3.9

#Dev ortamındaki gibi projeyi container a taşıdık. (.Dockerignore dakiler hariç.
COPY . /app

#testleri çalıştırdık.
RUN ng test --watch=false
RUN ng e2e --port 4202

#ng build komutu ile publish dosyalarını oluşturduk. dist klasörü altına.
RUN ng build --output-path=dist

#uygulamamızı yayınlamamız için nginx i yükledik.
FROM nginx:1.16.0-alpine

#dosyaları nginx e kopyaladık.
COPY --from=build /app/dist /usr/share/nginx/html

#nginx in 80 portunu dışarıya açtık.
EXPOSE 80

# nginx i çalıştırdık.
CMD ["nginx", "-g", "daemon off;"]
```
Yukarıda dikkat ederseniz iki adet FROM geçiyor, ilkinde node'dan miras alırken ikincisinde nginx den miras alıyoruz. Ayrıca node'dan miras alırken 'as build' şeklinde bir ibare var. Ardından COPY kısmında --from=build diye bir ibare var. Dockerda multi-stage denilen bir kavram vardır.

Şöyle anlatayım, docker olmadığını ve bu projeyi tamamladığımızı farz edelim. Sıra, build etme ve yayınlama aşamalarında. Build etmek için ihtiyacımız olanlar; angular/cli ve chrome; onun ihtiyacı olan ise node. Yani kısaca bunların olduğu bir ortama ihtiyacımız var; işte bu docker da bir stage anlamına geliyor, biz bunu yukarıda build stage olarak belirttik. Peki build ettik şimdi yayınlama yapacağız bunun için sadece yayınlama dosyalarına ve nginx'e ihtiyacımız var; bu da publish stage'imiz. Peki ortamları hazırladık kendi bilgisayarımızda build ettik, şimdi sıra publish dosyalarını yayınlama ortamına taşımaya geldi. Bunun için de docker da aşağıdaki komutu kullanıyoruz.
```shell
COPY --from=build /app/dist /usr/share/nginx/html
```
Kısaca Build stage'indeki /app/dist klasörü içindekileri şu an içinde bulunduğum (en son FROM ladığım) stage'deki /usr/share/nginx/html klaösürüne taşı.

Prod ortam için Dockerfile'ımız hazır. Ardıdan bunu build edip imaj haline getirmeliyiz. sampleangular:prod ibaresi ile aynı isimli docker imajının production olduğunu etiketliyoruz.
```shell
docker build -f Dockerfile-prod -t sampleangular:prod .
```
Aşağıdaki komut ile prod imajımızı çalıştırabiliriz. Ya da dockerHub'a gönderip oradan çalştıracağımız host'a çekebiliriz.
```shell
docker run -it -p 80:80 --rm sampleangular:prod
```
Güzel bir yazı oldu umarım faydalanırsınız.