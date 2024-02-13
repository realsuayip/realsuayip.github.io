+++
title = "Gerçek hayatta Django I: Sinsi ‘sinyal’ yürütümü"
date = 2023-11-14T07:02:03.347Z
tags = ["django", "turkish"]
slug = "django-sinsi-sinyaller"
+++

“Gerçek hayatta Django” olarak isimlendirdiğim yazı serisinin ilk yazısına hoş
geldiniz. Bu seride, _engin_ Django deneyimlerimi sizlerle paylaşıyorum.

İlk yazı olmasından ötürü ufak bir konsept tanımı yapacağım. Genel olarak,
Django’da yaptığım hataları, bunları nasıl çözdüğümü ve şöyle böyle kazandığım
tecrübeleri sizlere aktaracağım.

Ana amacım, Django ile fazla haşır neşir olmayan veya yeni başlayan
geliştiricilerin bu tarz yazıları okuyup “vay anasını” demelerini sağlamak,
mümkünse ekşi sözlükte
“[öğrenildiğinde ufku iki katına çıkaran şeyler](https://eksisozluk1923.com/ogrenildiginde-ufku-iki-katina-cikaran-seyler--2593151)”
başlığının altında bu yazıların linklerin paylaşılmasını sağlamak (hehe).

## Sinsi ‘sinyal’ yürütümü

Bu konuya ekşi sözlükten devam edelim, zira olay alakalı proje üzerinden
gerçekleşiyor.

Biliyorsunuz ki, ekşi sözlük’te bir çaylaklık sistemi var. Bu sistemde, 10
entry girerek bir sıraya alınıyorsunuz. Bir süre sonra entry’leriniz
inceleniyor; uygun görülürse yazar yapılıyorsunuz. Bu sistemin bir benzeri
de  [django-sozluk](https://github.com/realsuayip/django-sozluk)  projesinde
var.

Bizim senaryomuzda, çaylak bir kullanıcının tam 10 adet entry’si var. Kullanıcı
entry’lerinden birini silmek istiyor. Entry silindikten sonra kullanıcın toplam
entry sayısı 9'a düştüğü için yazar bekleme listesinden şutlanması gerekiyor.
Ama o da ne? Bunun için ayrılan bir logic olmasına rağmen, ve ilgili database
isteği de çalışmasına rağmen kişi listeden bir türlü çıkarılmıyor.

Aynı request’te çalışan diğer database isteklerine bakıyoruz. Bir diğer istek,
kullanıcı entry sildiği için karma puanını düşürüyor (bu olay ekşide yok
sanıyorum, amacı kullanıcıları içerik silmekten alıkoymak — öyle bi uydurdum).

Çaylak sırasından şutlama mekanizması bir sinyal sistemi üzerinden çalışıyor
(aslında `delete` metodu kullanılmış — ama fark etmiyor). Karma düşürme işlemi
ise view’de halledilmiş.

Peki hata neydi? Hata, karma puanı düşürme ve çaylak yapma database
isteklerinin farklı user instance’leri üzerinden yürütülmesi. Şöyle ki:

1. `entry.delete()` çağrılıyor, dolayısıyla ona bağlı sinyal çalışıyor,
   içeride `ForeginKey` ile ilişkilendirilmiş kullanıcı şu şekilde
   kaydediliyor (buradaki `self` entry instance’i oluyor):

   ```python
   if (
       self.author.is_novice
       and self.author.application_status == "PN"
       and self.author.entry_count < 10
   ):
       # If the entry count drops less than 10, remove user from novice lookup.
       self.author.application_status = "OH"
       self.author.application_date = None
       self.author.queue_priority = 0
       self.author.save()

   ...
   ```

2. Hemen ardından, `request.user` kullanılarak karma puanı düşürülüyor, (bu
   varsayım doğru, çünkü kullanıcılar sadece kendilerine ait entry’leri
   silebiliyorlar; bu durum önden bir izinle kontrol ediliyor):

   ```python
   ...

   entry.delete()

   if not entry.is_draft:
       # Deduct some karma upon entry deletion.
       request.user.karma = F("karma") - 1
       request.user.save()

   ...
   ```

Sonuçta `request.user` ve `entry.author` nesneleri birbirinden habersizler.
Dolayısıyla ikisi memory’de sadece ilk kez çekildikleri anki bilgiyi
tutuyorlar. Yani `request.user` kaydedildiği zaman memory’de hala çaylak
sırasında olduğu için, daha demin çaylak sırasından çıkardığımız adam tekrar
çaylak sırasına giriyor.

Bunun sebebi `save` metodunun varsayılan olarak BÜTÜN field’leri overwrite
etmesi. Bunu önüne geçmek için `update_fields` argümanı verilmeliydi. Bu
sayede sadece ilgili field’ler update edilirdi. Karma ve çaylaklık bilgileri de
kesişmediği için sorun olmazdı.

Bana kalırsa `update_fields` burada ikincil bir sorun. Esas sorun,
instance’lerin birbirinden bağımsız olduğunun unutulması. Dolayısıyla en
garantici yöntem, bütün operasyonları aynı instance üzerinden yapmak. Tabii
burda işin içinde sinyal olduğu için ne olduğu görmek ekstra zor, zira kod
lineer bir şekilde okunamıyor.

Bu durum özellikle aynı nesneye paralel bir şekilde update’ler yapıyorsanız çok
canınızı yakar; ama bu başka bir konu 🤓.

Temel sonuçlar şu şekilde:

1. Birden fazla operasyonun aynı anda bir nesneye uygulanması gerekiyorsa, her
   zaman bir instance üzerinden bunu yönetin.
2. Her zaman `update_fields` kullanın. Bu sayede potansiyel hatalardan
   kaçabilirsiniz (bonus: güzel bir database optimizasyonu).
3. Sinyal kullanırken bu gibi durumlar için ekstra dikkatli olun, mümkünse
   nesneler arası sinyaller birbirlerini değiştirmesinler. Hatta mümkünse hiç
   sinyal yazmayın; sinyal maintain etmek acı verir.

Bu sorunun çözüldüğü
[commit’i görmek için buraya tıklayın](https://github.com/realsuayip/django-sozluk/commit/8d7f4a60959dcce7d2c31bb2b5ef5d9fca808a8c).
