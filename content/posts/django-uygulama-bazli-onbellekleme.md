+++
title = """Django’da uygulama bazlı önbelleklemeyi “context processor”lar
bağlamında inceleyelim"""
date = 2021-06-27T18:23:50.080Z
tags = ["django", "turkish"]
slug = "django-uygulama-bazli-onbellekleme"
+++

Çok sofistike bir başlık oldu, ama basit bir konuya (ya da konunun basit
kısmına) değineceğiz: ön belleğe alma. Biliyorsunuz ki veri tabanı sorgularını
ne kadar optimize edersek edelim, bazen istediğimiz hızlara kavuşamıyoruz. Bu
durumda hızı arttırmak için iyi çözümlerden biri verinin dinamikliğini azaltma
tavizini vermek. Ön belleğe alma çok katmanlı bir yapı ve bulunan bağlama göre
ne yaptığınız (neyi nasıl ön belleğe aldığımız) değişir. Ben bu yazıda uygulama
bazında, yani tamamen sunucu kısmında yapacağımız ön belleğe alma işlemine
değineceğim. Nitekim, bu da tek katmanlı değil, istek (request) bazında ön
belleğe alma veya almama durumları da var, ikisine de bakacağız.

Şimdi, ön belleğe alma durumunu gerektirmek için bir senaryo uyduralım. Mesela
context processor yapısı ön belleğe alma işlemini yapmak için biçilmiş bir
kaftan. Bu yapıyı her request bağlamına bir şeyler eklemek istediğimiz
kullanıyoruz. Örneğin sitemizde kategoriler olsun ve sitenin header kısmında bu
kategorileri listeleyelim. Bu durumda her request’te veri tabanından bu
kategorileri çekmemiz gerekecek. Kategoriler de sık değişmeyen şeyler
olduklarından hemen bunları ön belleğe almak istiyoruz, zira her request’te 
veri tabanına fazladan bir istek atmak istemiyoruz, her ne kadar bu istek çok
hızlı çalışsa da. Örnek bir context processor:

```python
def categories(request):  
    return {"categories": Category.objects.all()
```

Şimdi bu yapıyı ön belleğe (bundan sonra cache diyelim) alalım. Bunu yapmak
için Django ile birlikte gelen cache backend’i kullanacağız. Kod üzerinden bu
api’yi açıklamak istiyorum:

```python
from django.core.cache import cache

def categories(request):  
    cache_key = "header_categories"  
    value = Category.objects.all()  
  
    if cached_value := cache.get(cache_key):  
        value = cached_value  
    else:  
        cache_timeout = 86400  
        cache.set(cache_key, value, cache_timeout)  
  
    return {"categories": value}
```

Öncelikle her cache bağlamında mantık şu şekilde işler:

1.  Belirlenen anahtar cache veri tabanında kayıtlı mı diye bakılır.
2.  Eğer kayıtlı ise (cache hit) bu kayıt döndürülür, yeni hesaplama yapılmaz.
3.  Eğer kayıtlı değilse (cache miss) belirlenen anahtarla cache veri tabanına
yeni kayıt hesaplanarak eklenir.

Örneğimizde  `cache_key`  hesaplanacak değerin bulunması için tutulan eşsiz bir
anahtar, bu anahtar sayesinde cache veri tabanına erişeceğiz. Yukarıda
anlattığım adımları `cache` nesnesini kullanarak `set` ve `get` metotlarıyla
basit bir şekilde gerçekleştirdim. Queryset’ler cache edildiği anda
hesaplandığı için cache veri tabanında bunun hesaplanmış hali tutuldu. Eğer
cache veri tabanında kayıt yoksa Queryset’in kendisini döndüm, bu durumda
yeniden hesaplanmasını template kısmına bırakmış oldum. Siz de `cache`
nesnesini kullanarak ve yukarıda 3 adımlı mantığı kullanarak istediğiniz her
şeyi cache’ye alabilirsiniz.

Eğer yukarıdaki örnekte Django admin panelinden yeni bir kategori ekleseydik
sitemizde hiçbir değişiklik olmazdı. Bunun arkasından dolaşmak için ilgili
modelin save metoduna `cache` nesnesinin `delete` metodunu kullanarak her
yeni kategori eklemede cache’yi temizleyebiliriz. Yine cache bağlamını
spesifikleştirmek için anahtarı kullanabileceğimizi anlamışsınızdır. Örneğin
giriş yapmış kullanıcı bazında bir cache istersek bu anahtarın içine kullanıcı
adını sıkıştırabiliriz.

Bu cache api’yi kullanmak, ardından biraz server overhead’i getiriyor.
Production’da cache database ile haberleşmek için bir cache backend’i
ayarlamanız lazım, bunun için hazır kütüphaneler var. Sistemde bir de cache
database olması gerekiyor, bunlardan en popüleri  [Redis](https://redis.io/).

---

Şimdi dilerseniz biraz da request bazlı caching’i anlatayım, spesifik olarak
“memoization” Diyelim ki user modeline bir method eklediniz ve her request’te
bu metodu birkaç farklı yerde birkaç kere kullanıyorsunuz, fakat bu durumda bu
metot birkaç kere çalışacak ve ağır bir hesaplama yapılacaksa bu boş yere
birden fazla kere yapılacak. Buradaki isteğimiz ilk hesaplanan değerin belleğe
alınması ve takip eden çağrıların bu değeri kullanması. İşte bu durumda 
[`cached_property`](https://docs.djangoproject.com/en/3.2/ref/utils/#django.utils.functional.cached_property) 
decorator’u biçilmiş kaftan, tam da bu dediğimizi yapıyor. Eğer bir property
metodu değil de normal bir metodu bu yöntemle cache’ye almak isterseniz yine 
Python standart kütüphanesinde bulunan `lru_cache` decorator’unu
kullanabilirsiniz. Bu mekanizmalar için bir cache veri tabanına gerek yok, zira
o anda bütün işlemler RAM’de gerçekleştiriliyor.

İşin ilginç kısmı, eğer uygulama bazlı caching yapıyorsanız, bu cache işleminin
yapıldığı yerde yine muhtemelen LRU cache’ye ihtiyacınız var zira benzer bir
şekilde cache veri tabanına boşuna yineleyen istekler atmak istemeyiz.

İşte cache yapısını kullanmak bu kadar basit. İşin sonunda birkaç metot ve
fonksiyon kullanarak uygulamanıza büyük bir hız avantajı kazandırabilirsiniz.
Bu metotların çeşitliliği Django ile artıyor tabii, `cache` nesnesinin 
etrafında toparlanmış bir sürü [wrapper fonksiyonlar](https://docs.djangoproject.com/en/3.2/topics/cache/#template-fragment-caching)
var ve örneğin view’lerinizi kolay bir şekilde cache etmenizi sağlıyor. Yine
template’lerin içinde de  `cache`  tag’i kullanabiliyorsunuz. Son olarak, eğer
sürekli `cache` nesnesi ile uğraşmak istemiyorsanız ve Django’nun sunduğu
wrapper’ler de pek işinize yaramıyorsa bu bahsettiğim 3 adımlık senaryoyu
gerçekleştiren bir decorator yazdım, ona da [buradan](https://github.com/realsuayip/django-sozluk/blob/36cdf3f10ed18d0a57f1420b0392cf15ef03985d/dictionary/utils/decorators.py#L9)
ulaşabilirsiniz.

Son olarak ufak bir bilgi, yazının en başında her request’te veri tabanına
istek atmanın istemediğimiz bir durum olduğunu bu sebeple caching yapmak
istediğimizi söylemiştim. E bildiğiniz gibi her request’te bir `request.user`
‘imiz var. Bu tabii havadan gelmiyor, Django her request’te mevcut kullanıcıyı
bulmak için session tablosuna istek atıyor. Session’a ait kullanıcı referansını
Django’nun sunduğu bir [backend](https://docs.djangoproject.com/en/3.2/topics/http/sessions/#using-cached-sessions)
ile cache’ye alabiliyoruz.
