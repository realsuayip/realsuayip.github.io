+++
title = "Django’da “bilet” tabanlı WebSocket kimlik doğrulama"
date = 2023-12-18T18:16:58.998Z
tags = ["django", "turkish"]
slug = "django-bilet-tabanli-websocket-authentication"
+++

Bu yazıda Türkçe’ye “bilet tabanlı kimlik doğrulaması” şeklinde çevirdiğim,
İngilizce’de “ticket-based authentication” olarak geçen, WebSocket bağlamında
kullanılan bir kimlik doğrulama yöntemini anlatacağım.

Eğer Django’da WebSocket uygulamaları yazdıysanız bilirsiniz ki bu işleri
halletmek için genelde `channels` kütüphanesinin kapısını çalarız,
dolayısıyla örnekleri bu kütüphane üzerinden göstereceğim. Yazıyı daha sade
tutmak için kütüphaneyi açıklamak için fazla zaman harcamayacağım; bildiğinizi
varsayıyorum.

Öncelikle "Channels bize zaten kimlik doğrulama yöntemi
veriyor, `scope["user"]` diye bir şey var, sen hayırdır?” diye soruyor
olabilirsiniz. Haklısınız, fakat burada kullanılan kimlik doğrulama yöntemi her
zaman işe yaramıyor.

Birincisi, bu yöntemi kullanmak için cookie’leri kullanmak zorundayız zira
channels burada Django session cookie üzerinden kimlik doğrulaması yapıyor.

Eğer Django’da herhangi bir API yazdıysanız, session üzerinden kimlik
doğrulamanın aslında o kadar da kullanılmadığını bilirsiniz. Genelde stateless
JWT tokenler veya duruma özel olarak üretilen başka stateful token’ler olur.

Böyle durumlarda cookie ile session doğrulaması yapamıyoruz. Aynı zamanda
browser bağlamı dışında client’leriniz de olabilir. Örneğin mobil uygulamanızın
WebSocket’le bağlantı kurmak için cookie ayarlaması garip kaçan bir durum.

Yine WebSocket’in authorization için client tarafında başka sorunlar var.
Örneğin normal HTTP istekleri gibi kafadan header belirleyemiyorsunuz (bunu
yapabilseydik, pat diye API tarafından verilen JWT token’i atardık).

Örneğin, JavaScript’te bağlanmak için sadece WebSocket adresini ve protokolünü
verebiliyorsunuz. React veya Vue gibi SPA framework’leri bağlamında bu durum
sorun yaratıyor.

> Not: Eğer cookie’leri frontend domain’i ile paylaşıyorsanız, standard cookie
> yöntemiyle kimlik doğrulama yapabilirsiniz (SPA olsun veya olmasın), zira
> tarayıcınız WebSocket bağlantısı açarken cookie’leri otomatik gönderiyor.

Bu sorunu çözmek için öncelikle WebSocket tarafına bir kimlik bilgisi
göndermemiz lazım, buna “bilet” diyeceğiz. Aslında authorization token’den bir
farkı yok. Sadece WebSocket bağlamı için özel olarak üreteceğiz. Biletimizi
kesmek için de bir API geliştirmesi yapmamız gerekiyor. Giriş yapmış olan
kullanıcılar, bu API’yı kullanarak WebSocket bağlantısı kurabilecekleri bir
bilet alacaklar.

Bu biletin oluşturulma ve saklanma(ma)sı konusunda iki farklı yöntem var.
Durumunuza göre bu yöntemleri değerlendirip sizin için uygun olanı seçmeniz
gerekiyor.

## Stateless Yöntem

Öncelike stateless yöntemden bahsedelim, yani server’de herhangi bir veri
tutmadığımız yöntem. Bu yöntemde aslında bir JWT token üretiyoruz. Bu token’in
içinde kullanıcıya ait bilgiler bulunuyor (mesela ID’si).

Server tarafında tuttuğumuz secret bir key sayesinde gelen veriyi güvenli bir
şekilde doğrulayabiliyoruz. Bunun için Django’da halihazırda bulunan `signing`
kütüphanelerini kullanabiliriz. Örneğin:

```python
def create_ticket(user):
    signer = signing.TimestampSigner()
    obj = {"id": user.pk, "uuid": user.uuid.hex}
    return signer.sign_object(obj)
```

Burada `TimestampSigner` kullanıyoruz zira bu biletin çok kısa bir süre için
geçerli olması gerekiyor (örneğin birkaç saniye). Aşağıdaki doğrulama
fonksiyonunda, `max_age` argümanını kullanarak bunu sağlayabiliriz:

```python
def verify_ticket(ticket, max_age):
    signer = signing.TimestampSigner()
    obj = signer.unsign_object(ticket, max_age=max_age)
    id, uuid = obj.get("id"), obj.get("uuid")
    return id, uuid
```

Biletimizin içinde tutulan veriler de oldukça önemli. Bilet içeriği sitenizde
üretilen diğer imzalı string’lerden farklı olmalı, aksi takdirde replay
attack’lere maruz kalma şansınız var. Öte yandan, bilet içeriğinizin public bir
bilgi olduğunu unutmayın, hiçbir şeyi şifrelemiyoruz burada.

Ben, işime geldiği için user’ın ID ve UUID bilgisini koydum burada. Ek bir
önlem olarak da kullanıcının bilet kestiği IP adresini buraya koyabilirsiniz.
Daha sonra, WebSocket tarafında kontrol yapılırken aynı IP adresinden mi
bağlanılmaya çalışıyor diye bakabilirsiniz. Bu sayede potansiyel bir MITM
saldırısı zorlaşır.

## Stateful Yöntem

Bu yöntemde bileti imzalama yoluyla vermek yerine, bir veritabanına koyuyoruz.
Bu bağlamda Redis gibi bir database kullanmak mantıklı zira biletlerimizin çok
uzun süreli yaşamaması gerekiyor. Yani biletler timeout’u düşük bir şekilde
cache’de duracaklar.

Stateless yönteme nispeten bu yöntemin overhead’i biraz daha fazla. Eğer
herhangi bir nedenle imzalama yapmak istemiyorsanız (ki imzalama yaparken
gerçekten dikkatli olmalısınız) ve kullanıcılarınıza “random” tokenler vermek
istiyorsanız bu yöntemi kullanabilirsiniz. Bu sayede biletiniz hassas bilgiler
içerir duruma da gelir.

Yine stateless yöntemde bahsettiğim IP önlemi, bu bağlamda da alınabilir.

Pekala, biletimizi kestik ve kullanıcıya verdik. Peki WebSocket bağlamında ne
yapacağız? Burada biletin gönderilmesi için iki yöntem var:

1. İlk bağlantıda query parameter olarak, path içinde göndermek. Eğer bilet
   doğrulanamazsa bağlanmayı reddetmek.
2. İlk bağlantıyı koşulsuz kabul etmek, WebSocket üzerinden atılacak ilk
   data’nın bilet olmasını beklemek, eğer bilet doğrulanamazsa, veya belli bir
   sürede gönderilmezse bağlantıyı kesmek.

Ben şahsen ilk yöntemi tercih ediyorum, zira hem sunucu hem de istemci
tarafında bu yolu izlemek en kolayı.

Evet, normal şartlarda query parameter üzerinden bu tarz bilgiler gönderirseniz
sopayla kovalarlar; fakat bu bağlamda o kadar problem değil, zira bilet oldukça
kısa bir süre için geçerli.

İlk yöntemi kullanarak yazılmış bir channels middleware’i şu şekilde:

```python
from django.core import signing
from django.http import QueryDict

from channels.security.websocket import WebsocketDenier


class QueryAuthMiddleware:
    def __init__(self, app):
        self.app = app

    async def __call__(self, scope, receive, send):
        query = QueryDict(scope["query_string"])
        ticket = query.get("ticket", "")
        try:
            user_id, uuid = verify_ticket(ticket, max_age=3)
            scope["user_id"] = user_id
            scope["user_uuid"] = uuid
        except signing.BadSignature:
            denier = WebsocketDenier()
            return await denier(scope, receive, send)
        return await self.app(scope, receive, send)
```

Bu middleware’i de bildiğiniz üzere channels router’inden ekleyebiliyoruz,
genelde `asgi.py` içinde:

```python
websocket = QueryAuthMiddleware(
    AllowedHostsOriginValidator(URLRouter(websocket_urls))
)

application = ProtocolTypeRouter(
    {"http": django_asgi_app, "websocket": websocket}
)
```

Özetle bu middleware, gelen bileti kontrol ediyor, eğer geçerliyse içerisindeki
bilgileri alıp consumer scope’ine paslıyor. Bu aşamada siz kullanıcıyı
database’den çekip izin kontrolü de yaptırabilirsiniz.

Eğer bilet geçersiz ise, WebSocket bağlantısı 403 dönüp kapatılıyor. Eğer her
consumer için kimlik doğrulaması gerekmiyorsa, burada scope’e kullanıcın
doğrulanmadığı bilgisini paslayarak da ilerleyebilirsiniz.

İşte bu kadar. Bu konuda daha fazla okuma için:

* <https://lucumr.pocoo.org/2012/9/24/websockets-101/>

* <https://devcenter.heroku.com/articles/websocket-security>

* <https://channels.readthedocs.io/en/latest/topics/authentication.html>
