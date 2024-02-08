+++
title = "Django'da ORM'a bir bakış"
date = 2021-04-26T12:27:21.721Z
tags = ["django", "turkish"]
slug = "django-orm-giris"
+++

{{< youtube IrMRSmoBJKU >}}

> **Bu sorgulardan hangisi daha hızlı? Python'da query processing yapmak
> memory'i ne kadar şişirir? Raw SQL code smell midir? Şuayip böyle query
> yazmayı nerden öğrendi? ORM ve query management ile ilgili her sorunun iyi
> bir cevabı var!**

Django’nun en can alıcı özelliklerinden biri de bildiğiniz üzere sunduğu ORM’u.
OOP ile kan kardeş olan Python (ki kendisi meta class’ler ile çok esnek yapılar
sağlıyor) bağlamında yazılmış olan bu ORM’u bir inceleyeyim, yazayım dedim.

> Şimdi abartmak gibi olmasın ama, Django’da ORM’u biliyorsanız kafadan
> Django’nun yüzde altmışını biliyorsunuz demektir. Zira bu devirde web
> çatıların görevi veri tabanı ve istemci arasında aracılık yapmaktan ibaret.
> Dolayısıyla Django öğrenirken terazinin ağır tarafını ORM’da tutmakta fayda
> var.

ORM ile ilk yolculuğumuz modellerden başlıyor. **Model** dediğimiz yapı veri
tabanında saklamak istediğimiz verilen bir  **suretini** ortaya koyuyor.
Tanımladığımız  her bir model veri tabanında bir tabloyu temsil ediyor, bu
model içinde tanımladığımız  her bir **field** ise bu tablodaki bir sütuna
işaret ediyor. Model yapılarını kullanarak aynı zamanda bu tabloya dair
**sorguları** (query)  gerçekleştiriyoruz. Her modelin aynı zamanda bir
**manager’ı** olmak zorunda, bu yapılarla modelimizle sorgu yaparken
işimizi kolaylaştıracak çeşitli  **method’lar** ekleyebiliyoruz. Bir de
**model instance**’larımız var ki bunlar da tablodaki bir satırı temsil ediyor,
benzer şekilde instance’lar için de işimizi kolaylaştıran metotlar
tanımlayabiliyoruz. Modellerimizin üstbilgileri (**meta**) var ki burada index,
constraint gibi tabloyu ilgilendiren bazı üstbilgilerle diğer Django
uygulamalarının kullanabileceği (örneğin gösterim adı) bilgiler bulunuyor.

Şimdi örnek bir model inceleyelim:

```python
class Artist(models.Model):  
    name = models.CharField(max_length=256)  
    surname = models.CharField(max_length=256)  
  
    birth_date = models.DateField()
```

Gördüğünüz üzere çok basit bir yapımız var. Bu yapı üzerinden bahsedilen
konseptleri bir inceleyelim. Öncelikle bu modelin tanımına bakarak suretinin ne
olduğunu çok kolay bir şekilde anlayabiliriz herhalde, deneyelim; veri
tabanında adı, soy ad ve doğum tarihi sütunlarının bulunacağı bir tablo
oluşmasını bekliyoruz. Fakat bu doğru değil, zira Django bizim bir primary key
ihtiyacımız olacağını sezerek buraya gizlice (klasik, eşsiz pozitif tam sayı
bir pk) onu yerleştiriyor. Bu ayrıntı dışında yaptığı başka birtakım hareketler
de var fakat onlar şimdilik bizi pek ilgilendirmiyor.

> Yapmak için çok iyi bir sebebiniz yoksa Django’nun verdiği primary key yapısı
> ile oynamayın ve elle pk tanımı yapmayın.

Buradan çıkarmamız gereken bir sonuç var. Demek ki Django
[model tanımını miras bırakmaya](https://docs.djangoproject.com/en/3.2/topics/db/models/#model-inheritance)
izin veriyor. Bu da sürekli kullanılan model yapılarını tekrar tekrar yazmamak
için güzel bir yol sunuyor. Ki pek çok Django projesinde bu özelliğin
oluşturulma ve değiştirilme tarihlerini tutan bir soyut model üzerinden pratik
edildiğini de gördüm.

## Field kavramı

Şimdi bir de field’lere bakalım. Burada üç tane tanımlamışız (ama açıklandığı
üzere aslında dört). Burada kullandığımız field’ler alışılagelmiş, sıkça
kullanılan field’ler. Django field seçenekleri bakımından çok cömert, hatta
ufak bir takım  _hack’ler_ ile dilediğiniz her şeyi field haline
getirebiliyorsunuz. Biz şimdilik
[elde ne varsa](https://docs.djangoproject.com/en/3.2/ref/models/fields/)
onla irade edelim. Gördüğünüz gibi field’lere keyword argument verebiliyoruz;
bunlar bu field’e dair üstbilgileri içeriyor. Üstbilgi konusunda burada Django
kendi çatısını da es geçmiyor ve yine çok işimize yarayacak olan
**`validators`** ve  **`verbose_name`** gibi çok önemli üstbilgilere de yer
veriyor.

> Field’leri isimlendirirken çok düşünün. Zira bunlar refactoring’i oldukça can
> sıkıcı şeyler. Örneğin  burada zaten artistten bahsettiğim için ad kısmını
> **name** olarak bıraktım, çoğu kişinin yaptığını gördüğüm hatalardan biri de
> böyle bir tabloda ad için **artist_name** isimli bir field oluşturmaları.

Field üstbilgilerinden önemli birkaçını zikredelim.  **`null`** field’in boş
kalması durumda veri tabanında `NULL` olarak temsil edilip edilmeyeceğine karar
veren bir argüman,  eğer belirtmezseniz `False` oluyor ki bu da bu field’in
asla boş kalmaması gerektiğine karar veriyor  (string’lere dair aşağıda bir
anekdot vereceğim).  **`blank`** app’ler için bir üstbilgi ve  admin sitesinde
(ki kendisi bir app’tir) bu field’in boş bırakılıp bırakılamayacağına karar
veriyor.  **`db_index`** bu field için bir index oluşturulup oluşturulmayacağına
karar veriyor. **`default`** varsayılan bir değer atamaya yarıyor.
**`unique`** bu field’in tablo boyunca eşsiz olup olmayacağını belirliyor.
**`validators`** bu field’in içeriğine ve doğruluğuna dair kontrolleri yapan
validator yapılarını (Django bu yapıları sunar) içeriyor; bu üstbilgi yine
sadece app’ler için; yani bu validator’ları ihlal eden verileri veri tabanına 
sokmak gayet mümkün.

> Yazı tabanlı field’lere (örneğin TextField, CharField ve EmailField gibi)
> **null** argümanı `False` olmalı (varsayılan hali böyledir). Aksi takdirde bu
> field’lerin boş değerleri için iki seçenek çıkıyor  `NULL` ve `""`
> (boş string).  Aynı şeyi ifade etmek için veri tabanında iki farklı değer
> tutulması istemediğimiz bir durum; bunun yüzünden ileride can sıkıcı
> hadiseler meydana gelmesini istemeyiz.

Bir de ilişkisel field’lerden bahsedelim. Örneğin `Song` isminde bir başka
modelimizin olduğunu varsayalım. Bu durumda artist ve şarkıları eşleştirmemiz
için ilişkisel bir field kullanmamız gerekecek. Bu bahsettiğimiz örnekte iki
senaryo var; her şarkının bir artisti olabilir ya da her artistin birkaç
şarkısı olabilir. Bu dediğim şeyler size aynı gelmiş, ya da mantıksız gelmiş
olabilir; bir de veri tabanı bağlamında bakalım:

a) Şarkı tablomuzda artist için bir referans (artist_id) verebiliriz.  
b) Şarkıları sahipsiz bırakıp, şarkı ve artist eşleşmesi yapan üçüncü bir tablo
oluşturabiliriz.

Muhtemelen çoğu kişi ilk senaryoyu daha mantıklı bulacaktır. İşte bu senaryo
için  `ForeignKey` kullanıyoruz; yani sahiplerin önem arz ettiği durumlar.
İkinci senaryoyu uygun görseydik  `ManyToManyField` (m2m) kullanırdık; yani
sahiplerin önemsiz olduğu durumlar. m2m yapısını çoğunlukla herkesin/her şeyin
sahip olabileceği durumlarda kullanıyoruz, mesela bir şarkıya ancak bir artist
sahip olabilir fakat o şarkının spesifik bir oynatma listesine ait olmasına
gerek yok, bu durumda  oynatma listesi ile şarkılar arasında kurulacak bir
bağda m2m uygun düşer. _(Gerçek hayat senaryosunda şarkılar ve artisti direkt
ilişkilendirmek doğru olmazdı; şarkıların sahipleri aslında albümlerdir ve
sanatçılar albümlere sahiptir. Aynı zamanda bir şarkıya birden fazla artist
de sahip olabilir, ama bu detayları örnek hatırına unutuverin)_

m2m yapısı anladığınız üzere üçüncü bir tabloya sebep oluyor. Bazen bu tabloya
da ek field eklemek isteyebiliyoruz. Aynı örnekten gidersek, şarkının oynatma
listesine eklenme tarihi bu field’lerden biri olabilir. Bu durumda modeli açık
açık tanımlamamız ve  **through** argümanı ile m2m’de belirtmemiz  gerekiyor.
Eğer model tanımını yaparken bu üçüncü tabloyu oluşturmayı atladıysanız daha
sonra da oluşturmanız mümkün. Bu üçüncü tabloyu oluşturmadığınız durumlarda
tablonun (auto-generated) model tanımına erişmeniz yine mümkün.

İlişkisel field’lerde atlanan bir diğer özellik ise  ters ilişkiler. Örneğin
bir artistin sahip olduğu şarkıların hepsini nasıl bulabiliriz? Şarkı modeli
ile inşa ettiğimiz bir query’de artist field’ini kullanarak süzebiliriz mesela.
Ters ilişkiler bunu yapmayı çok sade ve okunaklı hale getiriyor. Örneğin
elimizde bir artistin model instance’ı olsun, bu artistin tüm şarkıları
listelemek içinşöyle bir yapıyı kullanabiliriz:

```python
artist_obj.song_set.all()
```

Peki bu `song_set` neyin nesi? Biliyorsunuz ki her şarkının bir artisti var.
Django varsayılan davranışında ters ilişkiyi isimlendirmek için bu kaynak
modelin ismini alıyor ve sonuna  `_set`  ekliyor. Bu isimlendirme biçimi benim
hoşuma gitmiyor ve genelde o modelin çoğul hali olarak değiştiriyorum, bu
senaryoda `songs` yapardım mesela (bu değişiklik field’de **related_name** 
argümanı ile yapılıyor). Bu durumda `artist.songs.all()` hakikaten çok
açıklayıcı bir yapı oluyor.  Ters relation kullanmadan da şu şekilde aynı
sonuca varabilirdik:

```python
Song.objects.filter(artist=artist_obj).all()
```

> Ters ilişkilere isim verin, unutmayın ve kullanın.

Ek olarak bire bir ilişkiler  için  `OneToOneField`  var. Her ne zaman bir
model instance’ın bir başka model instance ile (ve bu ilişki eşsiz olmalı)
eşleşmesi gerekiyorsa bu yapıyı kullanmamız gerekiyor. Örneğin şarkımızın bir
müzik videosu olsaydı (bu video için de bir model gerektiğini varsayalım),
böyle bir yapı kullanabilirdik. Bu yapıya aynı zamanda  `ForeignKey` ile,
**unique** argümanına `True` vererek ulaşabiliyoruz. Fakat `OneToOneField`
ek olarak ters ilişki kullanıldığı zaman direkt model instance veriyor
(`ForeignKey`’de tek elemanlı `QuerySet`).

Bazen de  modeli kendisiyle ilişkilendirmek  isteyebiliyoruz. Örneğin her
artistin favori artistleri olduğunu varsayabiliriz (m2m).  Bu durumda argüman
olarak bir  model class'i yerine `"self"` (string olarak) vermemiz gerekiyor
. Burada bir ufak detay;  Django m2m field’ini varsayılan durumda simetrik
olarak kabul ediyor. Yani bizim örneğimizde  eğer bir artist başka bir artisti
favori artisti olarak eklerse, favori olarak eklenen artist de otomatik olarak
öncül artisti favori eklemiş oluyor. Bu durumun önüne geçmek için `symmetrical`
argümanını `False` olarak değiştirmeniz gerek.

İlişkisel field’lerin ve ters ilişkilerin isimlendirilmesi yine önemli bir
detay. Örneğin m2m field’lerı için çoğul yapılar kullanmakta fayda var. Yine
yapılan hatalardan biri bu field’lerin isimlerine  `_id`  takısı eklemek;
yapmayın.

## Modellerin metotları

Bazen model instance’lar üzerinden çeşitli hesaplamalar yapmak isteyebiliyoruz.
Örneğin artistin ad ve soy ad alanlarının birleşik bir şekilde gösterileceği
pek çok senaryo düşünülebilir. Bu durumda akla gelen ilk çözüm şu şekilde:

```python
"Artist: %s %s" % (artist.name, artist.surname) 
```

Gayet etkili ve güzel bir çözüm. Tek sıkıntısı, artistin ad ve soyadını
yazdırmak istediğimiz yerlerde tekrar tekrar bu yapıyı yazmak durumunda
kalmamız. Django’da modellerin metotları da tam buna çözüm oluyor, gelişigüzel
bir şekilde model instance’a metotlar ve nitelikler ekleyebiliyoruz:

```python
class Artist(models.Model):  
    name = models.CharField(max_length=256)  
    surname = models.CharField(max_length=256)  
  
    birth_date = models.DateField()  
  
    @property  
    def full_name(self):  
        return "Artist: %s %s" % (self.name, self.surname)
```

Bu uygulamadan sonra  `artist.full_name`  şeklinde bir yapıyı kullanarak
artistin ad ve soy ad bilgisine erişebiliyoruz. Burada kullandığım `property` 
built-in fonksiyon da önem arz ediyor, eğer yazdığınız metodun bir field gibi
, yani bir nitelik gibi erişilebilir olmasını istiyorsanız bu fonksiyonu
kullanmanız gerek; aksi durumda normal metot çağırma stilini (parantezler ile)
kullanmanız gerekiyor.  Hangisini kullanacağınıza pragmatik açıdan yaklaşarak
karar verebilirsiniz.

> Bazen bu özel property’leri oluştururken veri tabanına maliyetli bir takım
> (modelin kendi field’lerinden bağımsız da olabilir) çağrılar yapabiliyoruz.
> Böyle bir senaryoda her niteliğe erişimimizde bu çağrı tekrar yapılıyor; bu
> da gereksiz yavaşlığa sebep oluyor. Bunun önüne geçmek için  `property` 
> yerine Django’nun bize sunduğu  `cached_property`  fonksiyonunu kullanmamız 
> gerek. Bu sayede sonraki her çağırışta ilk çağrıda alınan değer kullanılıyor.

Kendinizin tanımladığı metotlar dışında bazı özel metotlar da var. Bunlardan
önemli olan bazıları `__str__`, `get_absolute_url`, `save`, `delete` ve
`clean`. İlki instance’ımızın temsil edecek bir string. İkincisi de eğer varsa
bu instance’a işaret eden bir URL; genelde burada Django’nun `reverse` 
fonksiyonunu kullanıyoruz. Bu iki metodun özel olmasının sebebi pek çok app’in
bu kalıbı takip etmesi.

`save` ve `delete` metotlarını, bu (adı üstünde) aksiyonlar gerçekleşmeden
önce veya sonra yapılmasını istediğimiz şeyler için kullanıyoruz (override
ediyoruz). Örneğin diğer field’lere bağlı olarak otomatik bir şekilde oluşacak
bir field’imiz var ise, burada onun değerini belirlemek akıllıca olacaktır.
Özellikle  `save`  metodunu kullanarak bir çeşit validation yapmak isteğiniz
olabilir, ama kesin tavsiyem işi buraya bırakmamanız ve request handling
kısmında bitirmeniz. Bir de, Django sinyallerde de benzer bir özellik sunuyor;
hangisinin hangi durumda daha kullanışlı olacağını sinyalleri konuşurken
zikredelim.

`clean` ise validation için ayrılmış bir metot. Daha önce bunun için
field’lerde validators argümanını görmüştük. Bu metodu ise field’ler arası
bağlantıları/ilişkileri doğrulamak için kullanabiliriz.

> `save` ve `delete` instance bazlı olduğu için toplu (bulk) işlemlerde
> çağrılmazlar (örn. `QuerySet.delete`, `QuerySet.update`). Her bir instance’ın
> işlenmesi gereken bir bağlamda tek tek bu metotları çağırmanız gerek. Bu
> yüzden bu metotları değiştirirken ileride yaşanabilecek toplu değişim
> senaryolarını düşünüp alternatif çözümleri elde bulundurmak şart.

Model metotlarını inceledikten sonra görüyoruz ki  instance’a dair her aksiyonu
model tanımında halledebiliyoruz (ki `QuerySet`‘lere dair olan her şeyi de
manager’lar yoluyla halledeceğiz). Bu da şu soruyu karşımıza çıkarıyor: “Ben
veri tabanı ile olan işlerimi model tanımında mı halletmeliyim yoksa `View`‘da
mı?”  Bu soruya dair çeşitli görüşler var, uzlaşma olacağını düşündüğüm görüş
ise şu: Eğer yazılacak olan logic birden fazla yerde kullanılıyorsa model
tanımında; sadece tek seferliğine, spesifik bir yerde, kullanılıyorsa (ve
bunun ileride değişeceği öngörülmüyor ise) `View`‘da yazılmalı.

## Create, Delete ve Update

Bu konular üstünde fazla durmak istemiyorum zira bunlar söz dizimi ezberleme
meselesi ve sorgularda olduğu kadar fazla içgörü gerektirmiyor. Model sınıfını
kullanarak hemen bir instance oluşturabiliyoruz, mesela  `Artist(name="Rick")`.
Bunu bir de veri tabanına işlememiz gerekiyor ki burada zaten tanıştığımız
`save` metodunu kullanabiliriz. Yine varsayılan manager’da `create` adlı bir
metot da mevcut. Güncelleme yapmak için elimizde olan instance’ın niteliklerini
(field’lere mahsus) normal bir Python nesnesinin niteliklerini değiştirir gibi
değiştirip daha sonra `save` metodunu çağırmak suretiyle yapıyoruz. Silme
işlemi için de yine bahsettiğimiz  `delete`metodu hazırda bekliyor. CUD
işlemlerinin hepsi için toplu (bulk) işlemler de mevcut, yani her zaman
instance’lar üzerinde çalışmak zorunda da değilsiniz; bunları yine `QuerySet`
api’sinde bulmak mümkün.

> Güncelleme yaparken `save` metodunun `update_fields` argümanını **her zaman**
> sağlamakta fayda var. Aksi halde güncellemediğiniz halde diğer field’lar da
> instance’da bulunan değerler ile update edilecek (aynı olsalar bile). Bu da
> çeşitli durumlarda (örn. aynı anda farklı instance’lar ile çalışıyorsanız
> —habersizlik yüzünden — ) saç baş yolduracak durumlar ortaya çıkarabiliyor.
> Hem bu alışkanlık performans açısından da daha olumlu sonuçlar almanıza
> yarayacaktır.

`save` ile ilgili bir diğer ayrıntı da, güncellenecek olan field’in
instance’ın kendi field’ine bağlı olduğu durumlarda ortaya çıkıyor. Örneğin bir
artistin şarkı sayısını belirten bir  `IntegerField`  olsun. Bir yerde bu şarkı
sayısını bir arttırmamız gerektiğini farz edelim, bu durumda şöyle bir yanlış
sıkça yapılıyor:

```python
artist.song_count += 1  
artist.save(update_fields=["song_count"])
```

Buradaki problem, yaptığımız işlemin veri tabanında bir karşılığı olmaması,
bu da bir [race condition](https://docs.djangoproject.com/en/3.1/ref/models/expressions/#avoiding-race-conditions-using-f)
oluşturabiliyor (yine instance’ların birbirinden habersiz kalmasıyla ilgili).
Böyle bağıl bir güncellemede bu durumun önüne geçmek için Django’nun
sağladığı `F` ifadesini kullanmamız gerek:

```python
artist.song_count = F("song_count") + 1  
artist.save(update_fields=["song_count"])
```

`F`  Django’da temel ve önemli ifadelerden biri. Bu ifadeyle ilgili daha
detaylı bir tarifi ileride yapacağız. Buradaki  görevi güncelleme yapacak veri
tabanı sorgusunu hazırlamak, yani `+=1` ifadesini bağıl field’e de işaret
ederek SQL’da mantıklı bir biçime çevirmek.

> Veri tabanında karşılığı olmayan bir instance’ın pk’si  `None`  olur. Eğer bu
> ayrıma ihtiyacınız varsa bu niteliğe bakabilirsiniz; bu özellikle oluşturma
> sırasında tek seferlik işlemler yapacağınız zaman işinize yarar ve `save` 
> metodunda kullanımı yaygındır (tabii ki super çağrılmadan önce).

Bildiğiniz üzere Django’da CUD işlemlerini neredeyse her zaman `Form` api’si
(veya bunun türevleri) üzerinden hallediyoruz.  `Form`  kullanmadığınız bir
bağlamda alışkanlıktan dolayı doğrulama işlemlerini (validation) es geçmeye
meyilli olabilirsiniz. Veri tabanında olsun, olmasın  bir instance’ın valid
olup olmadığı `full_clean` metodunu çağırarak anlayabilirsiniz. Eğer garantici
olmak istiyorsanız, bu metodu `save` metodunda çağırabilirsiniz, bu sayede her
zaman valid olan instance’lar veri tabanına gidecektir.

> İlişkisel bir field’i olan bir modelde bu field’e değer verirken ilişkili
> instance’ın kendisine ihtiyacınız yok; sadece bu instace’in pk’si olsa da
> yeter. Eğer  elinizde pk var ise, boş yere ilgili instance’a ulaşma çabasına
> girmemelisiniz.

Son olarak sinyaller (**signals**) için ufak bir parantez açalım, zira bu
sinyal yapısını genelde CUD bağlamında kullanıyoruz. Kayda değer sinyalleri
şu şekilde sıralayabiliriz: `pre_save`, `post_save`, `pre_delete`,
`post_delete` ve `m2m_changed`. Bu sinyaller adlarının belirttiği aksiyonlarda
çeşitli işlemler yapmamızı sağlıyor.

Örneğin bir artistin single (tek bir şarkı) yayınlama isteği olsun, fakat biz
sistemimizde şarkıları artistlerle değil albümlerle özdeşleştirmiş olalım. Bu
durumda tek bir şarkı eklendiğinde o şarkı için otomatik bir albüm oluşturacak
mekanizmayı sinyaller yoluyla kurabiliriz. Bu sayede tek bir şarkı ekleme
isteği için kullanıcı daha az bir çaba sarf etmiş olur.

Peki,  `save`  ve  `delete`  model metotlarından daha önce bahsetmiştik.
Bunlara denk gelen sinyaller de mevcut, hangisini kullanacağız? Bu da yine
tartışmalı konulardan, fakat büyük oranda uzlaşma şu yönde: eğer  yapılacak
işlem sadece model instace’in kendisini etkiliyorsa model metotları, aksi halde
(yani tablolar arası bir işlemde) sinyaller kullanılmalı.

## Sorgular

Django’da sorgu yapmak için metot zincirlemesi yapılıyor. Bu zincirin her
halkası da bir  `QuerySet`  tipinde bir nesne aslında. En basitinden belirli
modele ait tüm kayıtları listelemek için şu şekilde bir yapı kullanıyoruz:

```python
Artist.objects.all()
```

Çok kafa karıştırıcı değil, fakat  `objects`  niteliği biraz kafamızı
karıştırabilir. Neden  `Artist.all()`değil mesela? Daha önceden manager’ların
varlığından bahsetmiştik. İşte  `objects` de Django’nun bize verdiği
varsayılan bir manager.  İleride de kendi manager’larımızı oluştururken bu
sınıftan miras alarak yapacağız ki Django’nun sunduğu pek çok özelliği
kullanabilelim.

Yukarıdaki sorgudan sonra elde edeceğimiz  `QuerySet`  nesnesi içinde artist
modeline ait model instance’lar olacak;  `QuerySet`  de iterable bir nesne,
yani bir for döngüsü kullanarak her bir instance’a erişebilir, yukarıda
bahsettiğimiz yöntemleri uygulayabilirsiniz.

Eğer tüm satırlardan ziyade en üstte olan 10 artisti isteseydik, listelerden
aşina olduğumuz  _slicing_ söz dizimini kullanarak bunu yapabilirdik:

```python
Artist.objects.all()[:10]
```

Şimdi `filter`, `exclude` ve `lookup` kavramları üstünde duralım. Adları 
üstünde, `filter` ve `exclude` metotları field’lere bağlı olarak süzme
işlemleri yapıyor.  Örnek olarak soy adı Astley olan sanatçıların hepsini
getiren bir sorgu oluşturalım:

```python
Artist.objects.filter(surname="Astley")
```

Fakat burada hoşumuza gitmeyen bir şey var, soy adı yanlışlıkla “astley” olarak
girilmiş sanatçılar bu `QuerySet` nesnesine dahil olmayacak. Bu istemediğimiz
bir davranış, bunu önlemek için bir `lookup` kullanmamız gerek:

```python
Artist.objects.filter(surname__iexact="Astley")
```

`iexact`  lookup’ı  büyük-küçük harf ayrımını göz ardı ederek aynı değere sahip
mi diye bakıyor. Lookup’lar için genel söz dizimi şu şekilde:

```python
fieldismi__lookupismi="değer"
```

İlişkisel field’lerde tablolar arası süzme için de aynı söz dizimini
kullanıyoruz. Bu örnekte (artistinin soy adı Astley olan tüm şarkılar) hem
tablolar arası ilişki için bir lookup, hem de sütun için bir lookup var:

```python
Song.objects.filter(artist__surname__iexact="Astley") 
```

Eğer birden fazla şartımız varsa, bu metotlara istediğiniz kadar süzgeci
keyword argümanı olarak gönderebilirsiniz. Daha önce metot zincirlemesinden
bahsetmiştik, bunun için bir örnek yapalım. Adı Rick olmayan fakat soy adı
Astley olan tüm artistleri bulmak için:

```python
Artist.objects.filter(surname__iexact="astley")  
              .exclude(name__iexact="rick")
```

`filter` ve `exclude` metotlarını tekrar tekrar zincirlememekte fayda var,
zira her zincirleyişimizde  `QuerySet`  nesnesi “klonlanıyor”, bu da
performansa etki ediyor. Aynı zamanda  `filter`  zincirlemesi ile ilgili meşhur
da bir  _problemimiz_ var: Bu  metodu zincirlemek ile metot içine çok sayıda
argüman göndermek arasında ne fark var?  Örneğin şu iki sorguya bakalım:

```python
Artist.objects.filter(surname="Astley").filter(name="Rick") # 1  
Artist.objects.filter(surname="Astley", name="Rick") # 2
```

Görünüşe göre, örneğimizde birinci sorguda önce soy adı Astley olanları
buluyoruz, daha sonra da bu (bulunan) setten adı Rick olanları buluyoruz.
İkinci sorguda ise adı Rick, soy adı da Astley olan artistleri buluyoruz.

Django’nun da yaptığı hakikatten bu. Fakat  iş ilişkisel field’lerde değişiyor.
Eğer ilişkisel field’ler işin içine girerse, Django bu ardışık süzmeyi yapmak
yerine her bir filter metodu için ayrı bir değerlendirme yapıyor. Bu örnekteki
field’leri ilişkisel farz edelim, her artistin bir profili olsun ve bu bilgiler
o modelde bulunsun, bu durumda:

```python
Artist.objects.filter(profile__surname="Astley").filter(profile__name="Rick") #1
Artist.objects.filter(profile__surname="Astley", profile__name="Rick") # 2
```

İkinci durum yine tahmin ettiğimiz gibi olacak.  İlk sorguda ise ismi Rick 
**olanlarla birlikte** soy adı Astley olan artistleri içeren bir `QuerySet`
oluşturmuş olacaktık. İşte bu yüzden ilişkisel süzme yaparken `filter`
zincirliyorsanız [bu duruma](https://docs.djangoproject.com/en/3.2/topics/db/queries/#spanning-multi-valued-relationships) 
dikkat etmenizde fayda var.

Bir de `get` metoduna değinelim. Bu metot zincirin  **son** halkası olarak 
kullanılabiliyor ve `filter`‘de olduğu gibi keyword argümanları olarak
lookup’lar alıyor. Bu metot eşleşen nihai instance’ı döndürüyor. Eğer 
verdiğimiz kriterlere uygun birden fazla instance tespit edilirse
`MultipleObjectsReturned` hiç uygun instance bulunamazsa `DoesNotExist`
exception’ları raise ediliyor. Bunları yakalamak için model namespace’i
kullanabilirsiniz, örn. `Artist.DoesNotExist`.

`get`‘in pek bilinmeyen bir özelliği de tek elemanlı `QuerySet`‘teki elemana
ulaşmanıza yarıyor olması. Örneğin Rick Astley adında sadece bir sanatçı
olduğunu varsayalım:

```python
rick = Artist.objects.filter(name="Rick", surname="Astley").get()
```

Verilen durumda pek mantıklı bir kullanım değil elbette. Fakat ileride daha
esnek yapılara ihtiyacınız olacağından, bu özelliği göz önünde bulundurmak
önemli.

`QuerySet`  nesnesinin pek çok metodu var, bunların her birine tek tek
değinmeyeceğim; bunlar şimdilik en önemli olanlarıydı. Bu metotları
[Django dokümantasyonunda çarşaf çarşaf listelenmiş](https://docs.djangoproject.com/en/3.2/ref/models/querysets/#queryset-api)
olarak bulabilirsiniz. Her birine bakıp ne işe yaradığına, nasıl
kullanıldığına bakmakta kesin fayda var.

## QuerySet’lerin “tembelliği”

Django’da `QuerySet`‘ler yazıldığı anda veri tabanına istek göndermezler. 
`QuerySet`‘lerin veri tabanına istek attığı durumda bir bu  `QuerySet`
_evaluate_ edilmiş oluyor. Şimdi hangi durumlarda bunun gerçekleştiğini
görelim. Bu çözümlenme işlemini kullanıcıya bilgi göstereceğimiz son ana kadar
sarkıtmak istiyoruz, zira bellekte oradan buraya boş yere büyük bir Python
listesi (ya da tuple, ne severseniz) taşımanın bir manası yok. Buna ek olarak
aynı  `QuerySet`  nesnesini birden defa çözümlemek istemeyiz (ki yine Django
bunu engellemek için bir takım cache nitelikleri de geliştirmiş).

İşte şu durumlarda çözümlenme gerçekleşiyor:

a) Iteration yapıldığı zaman. Örneğin  `QuerySet`  nesnesini for loop
kullanarak gezerseniz, for loop’un başladığı satırda çözümlenecektir.  
b) list, tuple, len, bool, repr gibi metotları  `QuerySet`  üzerinde
kullanıldığı zaman.  `len`  ve  `bool`  özellikle parantez gerektiren kullanımlar.

> Eğer bir  `QuerySet`  nesnesinde kaç instance var merak ediyorsanız, bunu
> `count` metodu ile yapmalısınız. `QuerySet` nesnesinde hiç instance var mı
> diye kontrol etmek istiyorsanız, bunu  `exists`  metodu ile yapmalısınız.
> Bu metotları kullanmak (len ve bool’a nispeten) katbekat daha
> performanslıdır.

Örneğin şu kullanımdan kesinlikle uzak durun:

```python
qs = Artist.objects.filter(surname="Astley", name="Rick")
if qs:  
    print("Artist found!")
```

Bu kullanımda koşul bloğu  `QuerySet`  nesnesinin bool metodunu çağıracak,
dolayısıyla tüm  `QuerySet`  boş yere çözümlenmiş olacak, çünkü burada model
instance’larına dair bir kullanım yok;  **derdimiz varlık-yokluk**.

Bunlar dışında  `QuerySet`  nesneleri  _pickle’lamak_,  _cache’lamak_  ve slice
alırken step parametresi göndermek de çözümlenmeye yol açıyor. Fakat bu
yapıları kullanmaya başladığınız zaman zaten içgüdüsel olarak bunun
gerçekleşeceğini bileceksiniz, o yüzden daha detaya girmeye gerek görmüyorum.

## F

Bazen lookup yazarken değer kısmının instance’daki bir değere denk gelmesini
istiyoruz. Örneğin toplam şarkı sayısı doğduğu günün sayısal değerine eşit olan
artistleri bulmak istediğimizi varsayalım, bu durumda şunu yapmamız gerekirdi:

```python
Artist.objects.filter(birth_date__day=F("song_count"))
```

`F`’in görevi burada açık,  `QuerySet`‘deki instancelerin  **değeri** için bir
referans bırakıyor. Yani `F("song_count")` yazılan yere sanki
`artist.song_count` yazılmış gibi oluyor, tabii bu  `artist`  dinamik. Yine
`save` metodu üstünde duruken, bu ifadenin instance bazlı olarak da
kullanılabileceğini görmüştük. Aggregation yaparken de  `F`  sıklıkla
kullanacağımız ifadelerden biri olacak.

## Q ile kompleks sorgular

Şimdiye kadar sadece AND sorgularını yapmayı öğrendik. Daha kompleks sorgular
için bize  `Q`  nesnesi yardımcı olacak. Bu nesneyle beraber hem AND, hem OR
sorguları, bu sorguların tersleri ve kombine edilmiş hallerini
oluşturabiliyoruz. Örneğin adı Rick ya da soyadı Astley olan artistleri
süzmek için:

```python
Artist.objects.filter(Q(name="Rick") | Q(surname="Astley"))
```

Gördüğünüz gibi OR sorgusu yapmak için  `|`  operatörünü kullanıyoruz. Benzer
şekilde AND için  `&`  kullanabiliriz. Fakat  `&`  genelde kullanılmaz, zira
AND’lamak istediğimiz lookup’ı  `filter`‘e argüman olarak da verebiliriz; yine
de iç içe geçmiş lookuplar için bilmek faydalı. Olumsuzluk katmak için
`~` operatörünü kullanıyoruz. Şimdi şu örneğe bakalım:

```python
(Q(name="Rick") & ~Q(surname="Astley")) | Q(birth_date__year=1966)
```

Bu lookup’da da adı Rick olan ve soyadı Astley olmayan VEYA doğum yılı 1966
olan artistleri süzüyor. Her logical operatör kombinasyonunda olduğu gibi
burada da parantezler işi değiştirebilir. `Q` ile oluşturacağınız lookup’ları
bir değişkende tutup daha sonra bu değişkeni argüman olarak  `QuerySet`
metotlarına da gönderebiliyoruz. Bazen lookup’ların dinamik olmasını istiyoruz,
örneğin kullanıcının seçimine göre bir lookup’ı çıkarmak ya da eklemek
isteyebiliriz. Bu durumda `|=` ve `&=` operatörlerini kullanabilirsiniz.
Bunlar  `+=`, `-=`  operatörleri ile benzer yapıya sahipler. Yine  `Q`  
bağlamında `F` gibi ifadeler kullanmak mümkün.

## Aggregation işlemleri

Bazen de instance’ların toplu olarak bir araya geldiği zaman oluşturdukları
niteliklerle ilgileniyoruz. Mesela oluşturduğumuz sorguda ne kadar eleman var
diye merak ediyoruz, ki bunun  `count`  ile yapılacağını söylemiştik. Bu en
basit aggregation isteklerinden bir tanesi. Aggregation yapmak için Django’nun
sunduğu fonksiyonları kullanacağız, bunlardan önemli bir kaçını `Min`,
`Max`, `Count`, `Sum` ve `Avg` olarak sıralayabiliriz. Şimdi
`Avg` kullanarak bir aggregation yapalım:

```python
Artist.objects.aggregate(average_song_count=Avg("song_count"))
```

Bu aggregation’ın amacı tüm artistlerin ortalama şarkı sayısını bulmak.
Bu işlemi yapmak için  `QuerySet`  üzerinde bulunan  `aggregate`  metodunu
tercih ettim.  `aggregate`  yine  `get`  gibi zincirin son halkası olmak
zorunda, zira bu metot bir dictionary döndürüyor (yani  `QuerySet`  olduğu
yerde çözümleniyor).

Şarkı sayısı en fazla olan artistin kaç tane şarkısı olduğunu bulmak isteseydim
`Avg`  yerine  `Max`  kullanabilirdim. En fazla şarkı sayısı bilgisini 
kullanarak da  `song_count`  field’ini süzme yoluyla o artistin kendisine
de ulaşabilirdik. Yine bu fonksiyonları kombine ederek aritmetik işlemler de 
gerçekleştirebiliyoruz. Örneğin en düşük şarkı sayısı ile en fazla şarkı sayısı
arasındaki farkı `Max("song_count") - Min("song_count")` şeklinde bulabiliriz.

Aggregation yaptığımız bağlamlarda çoğunlukla ilişkisel field’ler ile
uğraşıyoruz. Bu durumu bir forum bağlamında inceleyelim. Bir forumda konular
olur ve bu konulara yorumlar gelir. Bu bağlamda en yaygın sorgulardan biri
konularla birlikte bu konulara gelen toplam yorum sayısını göstermektir. Şimdi
bunu nasıl yapacağımızı görelim (Bu sorguda `Comment` modelinin `Topic`‘e
`ForeignKey`  ile ilişkilendirdiğini varsayılıyor):

```python
Topic.objects.annotate(comment_count=Count("comment_set"))
```

`annotate`  yine zincirlenebilen bir  `QuerySet`  metodu, bize aliasing (takma
isim verme işlemi) sağlıyor ve bu tip aggregation senaryolarında sıkça
kullanılıyor. Bu metot ile takma isim verdiğimiz ifadeleri daha sonra aynı
field’lerde olduğu gibi  `F`  ile referans gösterebiliyoruz, aynı zamanda takma
isme bağlı süzme işlemleri gerçekleştirebiliyoruz. Bu bağlamda sadece yorum
yapılmış konuları listelemek isteseydik `filter` metodunu zincire ekleyip
`comment_count__gt=0`  şeklinde bir lookup argümanı kullanabilirdik.

Bazen de `Comment` bazlı bir süzme yapmak isteyebiliriz, o zaman önce bir
`filter`  zinciri ekleyip burada ilişkisel field’i kullanarak bir süzme 
yapabiliriz, fakat birden fazla aggregation yapacaksanız ve bu aggregation’lar 
farklı koşullar istiyorsa bu yöntemi işe yaramayacaktır; bunun için de 
aggregation fonksiyonunun `filter` argümanına `Q` nesneleri ile
oluşturduğumuz süzgeçleri gönderebiliriz; bu durumda yine `comment_set`
ön ekini kullanmak durumda olduğunuzu da ekleyeyim.

> Eğer takma isim verdiğiniz bir ifadeyi kullanıcıya gösterim için
> kullanmayacaksanız, yani sadece o sorgu için oluşturulmuş ise  `annotate`
> yerine `alias` metodunu kullanmalısınız. Bu sayede ufak bir performans
> kazanımınız olacaktır.

Bu bilgiler ışığında `song_count` isimli bir field’in `Artist` modelinde 
gereksiz olduğuna dair kanınızın oluşması gerek. Zira şarkıları ilişkisel bir
yapıyla artiste bağlayacağımız için bu tarz bir bilgiyi aggregation
fonksiyonları kullanarak kolayca elde edebiliriz. Eğer `song_count`
bilgisine sürekli ihtiyacımız olsaydı bunu model metodu olarak tanımlayıp, bu
metodun gövdesine de oluşturduğumuz aggregation ifadesini yazardık.

> Arka plandaki bazı çeşitli sebepler yüzünden (SQL’da subquery değil, join
> kullanılması) bir sorguda birden fazla aggregation yaparsanız bunların sonucu
> yanlış olacaktır. Bunun önüne geçmek için kullandığınız aggregation
> fonksiyonlarında  `distinct=True`  argümanını kullanmayı deneyebilirsiniz.

Peki ilişkisel bir bağlamda artistlerin ortalama şarkı sayısını nasıl bulurduk?
Bunu yapmak için öncelikle her artiste şarkı sayısını aggregate etmemiz ve daha
sonra bu aggregation değerini kullanarak (yani takma isim vermemiz gerekecek)
yine bir sefer daha ortalama için aggregation yapmamız gerek. Sonuçta ortaya
şöyle bir yapı çıkıyor:

```python
Artist.objects.annotate(song_count=Count("song_set"))  
              .aggregate(average_song_count=Avg("song_count"))
```

Şimdi biraz da `values` metodu üzerinde duralım. Normalde bu metot sizin
modelden field’leri seçip dictionary halde almanıza yarıyor. Fakat işin içine
aggregation girince bu metot SQL’da  `GROUP BY`  kısmına girecek ifadeleri
belirliyor. O yüzden  `values`  kullanırken bunu göz önünde bulundurmakta fayda
var. Örneğin şu sorguyu ele alalım:

```python
Artist.objects.values("name")  
              .annotate(song_count=Count("song_set"))
```

Bu sorguda `GROUP BY` ifadesinde `name` yer alacağı için artistler adlarına
göre gruplanacaklar. Bu da demek oluyor ki aynı isme sahip artistlerin şarkı
sayıları toplanmış bir şekilde gösterilecek (örn. adı Rick olanlar kümülatif
bir şekilde toplam kaç şarkı yapmış?).

Aggregation’lar için genel fikirler bu şekilde. Django dökümantasyonunda
[incelenmesini tavsiye ettiğim bir kopya kağıdı](https://docs.djangoproject.com/en/3.2/topics/db/aggregation/#cheat-sheet)
verilmiş. Bunu kullanarak neyin var/mümkün olup olmadığını hızlıca özümsemek
mümkün.

## **Manager’lar**

Öğrendiğimiz bilgiler doğrultusunda manager’lara ihtiyacımız olacak. Daha önce
manager’ların model bazında (instance bazında değil) işlemler konusunda bize
yardımcı olacaklarını söylemiştik. Örneğin sürekli olarak belirli tip artistler
üzerinden  `QuerySet`‘ler oluşturuyorsak, bu duruma özel bir kısayol
oluşturabiliriz. Şimdi örnek bir manager inceleyelim:

```python
class ArtistManager(models.Manager):  
    def popular(self):  
        return self.alias(song_hits=Sum("song_set__daily_hit"))  
                   .filter(song_hits__gt=500)
```

Bu manager’a  `popular`  isimli metot eklenmiş, bu sayede popüler artistleri
sorgulamak için bir kısayol oluşturulmuş.

Bu manager’ı modele kaydetmek için model gövdesine, tercihen field
tanımlarından hemen sonra `objects_custom = ArtistManager()` şeklinde bir yapı
yerleştirmemiz gerekiyor. Ben burada kendi oluşturduğum manager’ı
`objects_custom` namespace’inde kullanmak istediğim için öyle adlandırdım, siz 
dilediğiniz gibi adlandırabilirsiniz. Böylelikle bu manager’a ve içinde bulunan
metotlara  `Artist.objects_custom.filter(...)`  şeklinde ulaşılabilir. Bu
manager’da Django’nun bize sunduğu varsayılan manager’ı miras aldık, ki bu
zaten bildiğiniz gibi `objects` namespace’inde olan manager;
`objects = models.Manager()`. Dilerseniz bu namespace’i override edebilirsiniz,
ama bu pek tercih edilmez, zira projeye yeni katılacak biri için ekstra bir
zihinsel masraf yaratmış olursunuz, hele ki öncül  `QuerySet`‘i değiştiriyor
iseniz.

Bir parantez olarak da, ters ilişkilerde manager’ı belirtmenin şöyle bir yolu
var (belirtmediğiniz durumlarda  `models.Manager()`  kullanılır):

```python
artist.song_set(manager="objects_custom")
```

Manager’ın öncül `QuerySet` biçimine değiştirmek için `get_queryset` isimli
özel metodu override etmeniz gerekiyor, bu durumda manager’ı takip edecek ilk
`QuerySet`  metodunu ekstra bir custom metot çağırmadan kullanabilirsiniz.
Örneğin yukarıdaki manager’a alternatif olarak  `objects_popular`  isimli bir
manager oluşturup öncül  `QuerySet`  biçimini değiştirebilirdik:

```python
class ArtistManager(models.Manager):  
    def get_queryset(self):  
        return super().get_queryset()  
                      .alias(song_hits=Sum("song_set__daily_hit"))  
                      .filter(song_hits__gt=500)
```

Hangi yöntemi kullanacağınız sizin pragmatik seçimlerinize kalmış. Bir modele
istediğiniz kadar manager ekleyebilirsiniz ve bir manager’a istediğiniz kadar 
metot ekleyebilirsiniz. Mental yükü azaltmak adına manager’ları sınıflandırıp
çeşitli namespace’ler altında toplamak mantıklı olacaktır.

> Django varsayılan manager seçimini manager’ların model gövdesindeki sırasına
> göre yapıyor, o yüzden kendi manager’larınızı kaydederken ilk sıradaki
> manager’ın öncül  `QuerySet`  metodunu override etmemesi önemli, aksi 
> takdirde ileride istenmeyen problemlerle karşılaşabilirsiniz. Varsayılan
> manager’ı elle belirlemek için `Meta.default_manager_name` 
> kullanabilirsiniz.

## Sorgu optimizasyonu

Kompleks sorguların yanında, bunları optimize etmek de önemli. İstemediğimiz
hiçbir veriyi çekmek istemeyiz; veriyi yanlış işlemeyi de.  `QuerySet`‘lerin
tembelliği başlığı altında, yazılan sorgunun çözümlenmesinin kullanıcıya bilgi
gösterilecek son ana kadar sarkıtılması gerektiğini öğütlemiştik. Bu da aslında
Django’da sorgu optimizasyonda  **en temel**  kural.

Sorgu optimizasyonu yapmak için kalifiye bir kütüphanemiz var;
[django-debug-toolbar](https://github.com/jazzband/django-debug-toolbar/).
Bu kütüphaneye kullanarak hangi sorgu hangi SQL’i oluşturmuş, ne kadar sürmüş
ve toplam kaç tane sorgu yapılmış gibi birtakım bilgilere erişebiliyoruz. Bunun
yanında yazdığınız bir query’nin oluşturduğu SQL sorgusunu görmek için
`str(QuerySet.query)` yapısını kullanabilirsiniz.

Bu konulu yazılarda adettendir, `prefetch_related` ve  `select_related`
baş köşeye konur, ben de öyle yapacağım. Bir sorgu yaptığınız zaman (aksini
belirtmedikçe) Django ilgili model’deki  **her** field’i çeker, ama ilişkisel
field’leri çekmez (daha doğrusu ilişkisel instance’ı), zira veri tabanı
bağlamında o sadece bir sayıdır; ilişkideki instance’a ulaşmak için ayrı bir
sorgu yapılması gerekir. Eğer o field’i illa isterseniz Django da zaten bunu 
yapar. Örneğin bir oynatma listemiz olsun, bu oynatma listesinde şarkılar m2m
ile ilişkilendirilmiş olsun. Oynatma listesinin detay sayfasında listenin
kendisiyle alakalı bilgilerle (yani ilişkisel olmayan field’ler) her bir 
şarkıya dair bilgileri göstermek isteriz. Bu da 100 şarkılık bir oynatma 
listesi için 101 tane sorgu yapılması anlamına gelir, zira Django her bir şarkı 
için ayrı sorgu yapar, bu da canımızı sıkar tabii. İşte `select_related` ve
`prefetch_related`,  `QuerySet`‘e benim bu ilişkisel verilere de ihtiyacım var
demenin bir yolu.

> Prefetch edilmemişse bile ilişkili instance’ın id’sine  `obj.relatedfield_id` 
> şeklinde ulaşabilirsiniz. Eğer  `obj.relatedfield.id`  kullanırsanız bu,
> bahsettiğimiz sebeplerden dolayı ekstra bir sorguya sebep olacaktır.

Bu metotların farkları, oluşturulan ilişkiye göre değişiyor. `prefetch_related`
m2m senaryolarında uygunken  `select_related`  `ForeignKey`  senaryolarında
uygun düşüyor. Bunları  `QuerySet`  zincirine ekliyoruz, argüman olarak
ilişkisel field’in ismini alıyorlar, aynı zamanda  **ilişkinin ilişkisini**
(bu prefetch yaparken istenen bir durumdur) de takip edebiliyorlar.
`prefetch_related` için aynı zamanda yapılacak sorguyu `Prefetch` nesnesini
kullanarak belirleyebiliyoruz.

Django’nun varsayılan olarak her field’i çekmesinin pek de elverişli olmadığını
idrak etmişsinizdir. Bazen 30 field’den sadece 5 tanesine ihtiyacımız olabilir.
Bu durumlar için de çeşitli metotlarımız var. `values` ve `values_list` field
seçmemize yarıyor, aynı zamanda  `QuerySet`‘i sırasıyla dictionary ve list
formatına dönüştürüyor fakat eğer model metotlarını kullanacaksak bu durumu
istemeyiz.

Bu bağlamda  `only`  ve  `defer`  metotlarını incelememiz iyi olur. Bunlar
maşayla tutulması gereken metotlar; yanlış ve dikkatsiz kullanımda optimizasyon
yapacağım diye sizi zarara sokabilir. Bu metotlar da field seçmenize yarıyor,
fakat olur da seçmediğiniz bir field’i çağırırsanız Django bu field’ler için
ayrı sorgular oluşturuyor; bu da potansiyel bir hatada yüzlerce sorguya yol
açabilir. `values` ve `values_list` için böyle bir durum söz konusu değil
zira kendileri dictionary ve list nesnelerine dönüştükleri için artık veri
tabanı ile bir ilişkileri kalmıyor.

Eğer `QuerySet` bir kere çözümlenecekse `exists` ve `count` metotlarını ayrı
çağırmayın, çözümlenmiş nesne üzerinde  `bool`  ve  `len`  metotlarını
kullanabilirsiniz, zira Django bunları cache’liyor. Template’lerde yine bu
kullanımını  `with`  etiketini kullanarak taklit edebilirsiniz.

Toplu işlemler için, Django’nun sunduğu mekanizmaları kullanın. Eğer for
loop ile bir  `QuerySet`‘i iterate ediyorsanız, kendinize bu işlemin Django
metotları kullanılarak toplu olarak yapılıp yapılamayacağını sorun.

> Emin olmak için her zaman profiling yapın, django-debug-toolbar can
> kardeşiniz olsun.

Bunlar Django tarafında bilmeniz gerekenlerden bazılarıydı, veri tabanı 
kısmında da yapmanız gereken birtakım şeyler de var; index oluşturulacak
field’leri belirlemek gibi, fakat bunlar bu yazının kapsamında değil.

İşte bu kadar. Django’nun ORM’una kısık gözle ufak bir bakış attık. Eğer buraya
kadar kesintisiz okuduysanız sizi tebrik ederim. Umarım okuyan herkes
kullanabileceği birkaç bir şey bulur, ya da en azından halihazırdaki
bilgilerini tazelemiş olur. Esenlikler.
