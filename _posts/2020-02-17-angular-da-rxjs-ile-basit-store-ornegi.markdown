---
layout: post
title:  "Angular'da RxJs ile Basit Store Örneği"
description: 
date:   2020-02-17 13:15:36 +0530
categories: Typescript Angular RxJs
---
Angular projelerinde componentler arası iletişimi Input ve Output lar ile sağlıyoruz. Fakat İç içe 3 4 ve daha fazla kırılımlı componentler var ise input ve output lar karmaşaya yol açabilir. Input ve Output yerine RxJs nesnelerini componentlere inject ederek bu karmaşayı çözebiliriz. 

Aşağıdaki adreste RxJs ile componentler arası iletişimin bir örneği bulunmakta.
https://github.com/ibrahimdal/SampleStoreForAngular9

Angular Cli ile standart bir Angular projesi oluşturalım.
```shell
ng new my-dream-app
```
Projemizde bir AppComponent ve onun içinde birden fazla CardComponent bulunacaktır. Genel yapı aşağıdaki şekilde.
*app.component.html
```html
<app-card [no]="1"></app-card>
<app-card [no]="2"></app-card>
<app-card [no]="3"></app-card>
<app-card [no]="4"></app-card>

<div class="appWrap" *ngIf="lastPushedButtonNo>-1">
  Pushed Last Button No :  {{lastPushedButtonNo}}
</div>
```
*/card/card.component.html
```html
<div class="cardWrap">
    <button (click)="onPushCard()">Card {{no}}</button>
</div>
```
RxJs kütüphanesine ait Subject classını kullanacağız. Subject classı Generic bir class olup belirttiğimiz class tipinde nesnelerin bir uç tan bir uca taşınmasını sağlar. Yani aynı Subject nesnesini bir componentten next lerken diğer component ten subcribe edebiliyoruz. Bu sayede next leme sırasında gönderdiğimiz data o Subject'i dinleyen tüm componentlere ulaşıyor.

Önce bu Subject'i içinde barından appstore.service.ts adında bir class oluşturalım.

*/store/appstore.service.ts
```javascript
import { Injectable } from '@angular/core';
import { Subject } from 'rxjs';

@Injectable()
export class AppStoreService {
     private rx:Subject<AppStoreModel> = new Subject<AppStoreModel>();

     next(appStoreModel : AppStoreModel){
        this.rx.next(appStoreModel);
     }

     subscribe(fonk){
         return this.rx.subscribe(fonk);
     }
}
```
AppStore servisi Injectable olarak belirttik bu sayede onu provider olduğu tüm tüm modüllerde kullanılan componentlere inject edebileceğiz.
Diğer konu AppStore servisinin içinde 'Subject' nesnesi bulunuyor ve bu nesne 'AppStoreModel' clasını tutuyor.
AppStore'un 2 adet fonksiyonu var; next: veriyi Subject içine göndermek için, subscribe: gönderilen verinin alınması için dinleme işlemini yapıyor. subscribe içine gönderdiğimiz fonksiyon ile veri geldiğinde nasıl bir davranış gerçekleştireceğini belirtiyoruz.
```javascript
export class AppStoreModel {
    action:AppStoreAction;
    payload:AppStorePayloadModel;
}

export enum AppStoreAction{
    UNPUSHED = 1
}

export class AppStorePayloadModel {
    no : number;
}
```
AppStoreModel model clasının iki property si var. action: yapılacak işlem, payload: işlemde kullanılacak veri.
AppStoreAction ile işlemleri bir enum da belirttik. AppStorePayloadModel de bizim veri tipimiz.

Öncelikle AppStore bir provider olduğu için bunu AppModule'e kaydedelim.
```javascript
providers: [AppStoreService],
```

Ardından CardComponent i oluşturalım. (card.component.html i yazının başında vermiştim.)
*card/card.component.ts
```javascript
import { Component, Input } from '@angular/core';
import { AppStoreService, AppStoreModel, AppStoreAction, AppStorePayloadModel } from '../store/appstore.service';

@Component({
    selector:'app-card',
    templateUrl:'./card.component.html',
    styleUrls:['./card.component.css']
})
export  class CardComponent {

    @Input('no') no : number;

    constructor(
        private store : AppStoreService
    ) {
    }

    onPushCard(){
        this.store.next(<AppStoreModel>{
            action : AppStoreAction.UNPUSHED,
            payload : <AppStorePayloadModel>{
                no : this.no
            }
        });
    }
}
```

Card component dışardan Input ile bir numara alıyor. İçindeki butona basıldığında AppStore'a numaranın değerini ve
yapılan action'ı bildiriyor. Burada store'u dinleme işlemi bulunmuyor.

NOT: CardComponent'i app.module'e declare etmeyi unutmayın.

*app.component.ts
```javascript
import { Component, OnDestroy, OnInit } from '@angular/core';
import { AppStoreService, AppStoreModel, AppStoreAction } from './store/appstore.service';
import { Subscription } from 'rxjs';

@Component({
  selector: 'app-root',
  templateUrl: './app.component.html',
  styleUrls: ['./app.component.css']
})
export class AppComponent implements OnInit,OnDestroy {
 
  lastPushedButtonNo : number = -1;
  subStore : Subscription;
  constructor(
    private store : AppStoreService
  ) {}

  ngOnInit(): void {
    this.subStore = this.store.subscribe(
      (data:AppStoreModel) => {
        if(data.action == AppStoreAction.UNPUSHED){
          this.lastPushedButtonNo = data.payload.no;
        }
      }
    )
  }
  ngOnDestroy(): void {
    this.subStore.unsubscribe();
  }
}
```
AppComponent içinde store dinleniyor ve arayüzde basılan card numarası gösteriliyor. Ayrıca store dinlendiğinde bir subscription oluşuyor. Subscription lar dispose edilebilen nesnelerdir bu yüzden ihtiyacımız kalmadığında kapatılmalıdır. ngOnDestroy içinde bu kapatma işlemini görebiliriz.