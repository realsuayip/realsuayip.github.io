+++
title = "Bir Django projesinin yapısı"
date = 2022-12-11T11:34:29.477Z
tags = ["django", "turkish"]
slug = "django-proje-yapisi"
+++

Django’yu öğrenmek isteyenlerin kafasını karıştıran konseptlerden biri, basit
bir Django projesinin hiyerarşik yapısı ve bunun nasıl çalıştığıdır. Bir Django
projesi başlatmak için önce sanal bir ortam oluşturun, bu ortama Django’yu
yükleyin, ardından şu komutu çalıştırın:

```shell
django-admin startproject myproject
```

Bu komutu çalıştırdığınız dizinde `myproject` isimli bir klasör oluşacaktır.
Bu klasöre girerseniz ana dizinde `manage.py` isimli bir dosya göreceksiniz.
Bu dosyayı kullanarak Django'yu ayağa kaldırabilirsiniz:

```shell
python manage.py runserver
```

Eğer Django’nun komut satırında size verdiği adrese giderseniz, yerel
ortamınızda çalışan, Django tarafından hazırlanmış bir web sayfası
göreceksiniz.

Eğer oluşturulan dosya yapısını incelerseniz, Django’nun sırf bunu yapabilmesi
için tam  **altı**  dosya ve kök dizin dahil 2 dizin oluşturduğunu
göreceksiniz. İşin ilginç tarafı, henüz ekrana istediğimiz yazıyı bile
bastıramadık, bunu yapmak için birkaç dosya daha oluşturmamız gerekecek.

Şimdi kolları sıvayıp bir merhaba dünya çıktısı almaya çalışalım. Bunu yapmak
için önce projemizde bir ‘app’ (uygulama) oluşturmamız gerekiyor. Bu aşamada
app’leri Python modülleri olarak düşünebilirsiniz. Yani merhaba dünya çıktısı
vermekle görevli bir modül oluşturmamız gerekiyor. Bunu yapmak için şu komutu
proje ana dizininde çalıştırın:

```shell
python manage.py startapp helloworld
```

Bu komutu çalıştırdıktan sonra, `manage.py` dosyası ile aynı
seviyede `helloword` isimli bir dizin oluşmuş olacak. Bu dizinin içinde yine
**yedi**  tane yeni Python dosyası oluşacak. Yeter mi? Yetmez, zira Django bir
app modülü içinde `urls.py` dosyasını oluşturmuyor, bu dosyayı da elle
oluşturmak gerekiyor; o zahmete ve gerekliliğinin sebebine şimdilik girmeyelim.

Merhaba dünya serüvenini sona erdirmek üzere, `views.py` dosyasında şu yapıyı
oluşturun:

```python
from django.http import HttpResponse


def index(request):
    return HttpResponse("Hello World!")
```

E tabii bu mesajın hangi URL’den verileceğini belirtmemiz gerekiyor (zira
yukarıda böyle bir bilgi yok fark ettiyseniz). Bunun için de, `myproject`
dizini içindeki `urls.py` içine de bir referans eklememiz gerekiyor:

```python
from django.contrib import admin
from django.urls import path

from helloworld.views import index

urlpatterns = [
    path("", index),
    path("admin/", admin.site.urls),
]
```

Şimdi tekrar yerel ortamı ziyaret ederseniz ‘Hello World’ yazısının bizi
karşıladığını göreceksiniz.

Bu aşamada “Yahu altı üstü bir merhaba dünya için bu kadar zahmete (4 dizin, 13
dosya) gerek var mıydı? Dosyaların çoğunu kullanmadık bile?” demenizi
bekliyorum. Eğer önceden Flask veya Express gibi microframework’lar
kullandıysanız tek dosyada bu işin hallolduğunu bilirsiniz.

Aslında bu örnek için bu dosyaların çoğuna ihtiyacımız yok, hatta hepsini silip
tek dosyada da işinizi halledebilirsiniz. Mesela şu kodu bir Python dosyasına
yazıp, komut satırından yine `runserver` diyerek çalıştırabilirsiniz:

```python
import sys

from django.conf import settings
from django.http import HttpResponse
from django.urls import path

settings.configure(DEBUG=True, ROOT_URLCONF=__name__)


def index(request):
    return HttpResponse("Hello World!")


urlpatterns = [path("", index)]

if __name__ == "__main__":
    from django.core.management import execute_from_command_line

    execute_from_command_line(sys.argv)
```

> Gerçek hayatta denemeyiniz.

Bu aşamada bu dosyaların gerekliliğini anlamak için Django’nun geliştirilme
felsefesini anlamamız gerekiyor.

## Monolithic (yekpare) dizayn ve “Batteries included” felsefesi

Bir backend web framework’tan minimum beklentimiz, istediğimiz URL’leri
tanımlayabilmek ve bu URL’leri bir takım fonksiyonlarla ilişkilendirmektir.
Bunu yaparken de çeşitli HTTP yapılarının soyutlanmış bir şekilde elimize
ulaşmasını isteriz; örneğin request header’lerini kolayca bulabilmek gibi.

Hakikaten de, çoğu web framework bunu sağlar, fakat daha kompleks işler yapmak
istediğiniz zaman üçüncü parti bir kütüphane kullanmanız gerekir. Örneğin
database bağlantısı yapmak ve tablolar oluşturmak için muhtemelen bir ORM
kütüphanesi kurmanız gerekir.

Diyelim ki ORM kütüphanesini kurdunuz, bu sefer de basit kullanıcı
giriş/çıkış/oturum işlemleri yapmak istiyorsunuz; e bunu yapmak için de bir
başka kütüphaneye ihtiyacınız var.

Kullanıcı giriş çıkış işlemi güvenlik konusunda pür dikkat gerektirir o yüzden
yine üçüncü parti bir güvenlik kütüphanesi de kurmanız gerekti. Tam bunları
yaparken fark ettiniz ki, kullanıcıdan aldığınız veriyi hem sanitize etmeniz
hem de doğrulamanız gerekiyor, o yüzden bir tane de validation kütüphanesi
kurdunuz.

İşte Django burada diyor ki: kardeş sen bu kütüphanelerle hiç uğraşma, ben
bunların hepsini ve daha fazlasını senin için yaptım; sen zaten birbirine
yamaladığın kütüphanelerle muhtemelen daha kötü bir stack oluşturdun. Sen benim
hazır kodumu kullan; git bussiness logic’ini yaz.

Nedir mesela Django’nun verdiği şeyler? En temelinde kocaman bir ORM veriyor
Django. Öyle ki bu ORM Django’nun neredeyse yarısını oluşturuyor, zira bütün
workflow’lar (kullanıcı işlemleri, validation, admin integration) ORM’un
etrafında tasarlanmış.

Tabii ki her use case için bu mükemmel bir tarif değil bu, bazen gerçekten de o
kadar soyutlama katmanına ve kütüphaneye gerek duymuyoruz. Bazen de gerçekten
kendi stack’imizi oluşturmak, özgür olmak istiyoruz.

> Bu demek değil ki Django bize özgürlük tanımıyor, eğer beğenmediğiniz bir
> yapı varsa bunu çıkartıp yerine üçüncü bir parti kütüphane alabiliyorsunuz.
> Her şey app dediğimiz plug-in sistemine bağlı.

Django’nun bu felsefesini biraz irdelediğimize göre, gelin şu başlangıç
projesinde oluşturulan yapıya, ve bu yapıdaki dosyaların ne işe yaradığına göz
atalım. Proje yapısı şu an da böyle gözüküyor:

```text
myproject/
├── helloworld
│   ├── __init__.py
│   ├── admin.py
│   ├── apps.py
│   ├── migrations
│   │   └── __init__.py
│   ├── models.py
│   ├── tests.py
│   └── views.py
├── manage.py
└── myproject
    ├── __init__.py
    ├── asgi.py
    ├── settings.py
    ├── urls.py
    └── wsgi.py

3 directories, 14 files
```

En temel olarak `manage.py` dosyasından başlayalım; bu dosya Django'yu serve
etmek için gereken management komutunu çalıştırmaya yarıyor, aynı zamanda
Django'nun pek çok command line gereçlerini bu modül yoluyla çağırabiliyoruz.
Mesela `runserver` ve `startapp` komutlarını yukarıda görmüştünüz.

`myproject` dizinini inceleyecek olursak, `settings.py` Django uygulamamıza
ait tüm ayarları içeren bir dosya, bunda hem kendimizin ürettiği hem de
Django'ya ve diğer üçüncü parti kütüphanelere ait ayarları
yönetebiliyoruz. `urls.py` projenin ana (root) URL dağıtımını gerçekleştiren
modül. Burada belirlediğimiz namespace'lere (yani app'lerimize) URL ataması
yapabiliyoruz.

`asgi.py` ve `wsgi.py` Django projemizi deploy ederken gereken olan
modüller; özetle bu modüller Python uygulamaları ile web serverlerin iletişim
kurabilmesi için oluşturulmuş bir standardı yerine getirmek için varlar.

Şimdi `helloword` app'imize bir bakalım.

* `apps.py` Bu dosya, app'e ait ayarları ve yapılandırmayı içeriyor. Burada
  app'imizin okunabilir ismini ve kod adını belirleyebiliriz.

* `models.py` Bu app'e ait modelleri yani database tablolarının tanımlarını
  içeren modül; app'i yeni oluşturduğumuz için diğer tüm modüller gibi burası
  boş. `migrations` klasörü, model eklediğiniz takdirde, database şemasının
  zamanla değişimini tutuyor bu sayede database'i olan gelişmeleri ileri ve
  geriye dönük olarak takip edebiliyoruz.

* `tests.py` bu uygulamaya ait testlerini içeriyor.

* `views.py` bu uygulamaya ait request karşılayıp response dönecek
  fonksiyonları içeriyor; yani kullanıcıları karşılayacak logic'iniz buradan
  başlıyor.

* `admin.py` Django'nun admin sayfasında gözükecek modellerinizi ayarlamak
  için kullanacağınız modül burası.

App’leriniz için bu modüllerin çoğunu kaldırabilirsiniz, mesela app’iniz model
veya view içeriyor olmak zorunda değil. Elinizde bir `apps.py` olması
yeterli.

İşte bir Django projesinin yapısı böyle. Modülleri oradan buraya taşıma
imkanımız da var, illa ki bu yapıda kalmak durumunda değilsiniz, fakat eninde
sonunda buna benzer bir yapıya kavuşuyorsunuz. Django geliştiricileri bu yapıyı
bize sunmuşlar, ister kullanın ister kullanmayın; internette pek çok farklı
Django proje yapısı bulabilirsiniz.

Ben şahsen bu yapıda `myproject` içindeki dosyaları bir üst dizine, app'lerle
aynı seviyeye taşıyorum; bunların ayrı bir klasörde durması garip geliyor.
