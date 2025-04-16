+++
title = "Async Python'u bir de benden dinleyin"
date = 2025-04-16T21:56:57Z
tags = ["python", "turkish"]
slug = "async-python-bir-de-benden-dinleyin"
+++

Çalışan fonksiyonların birbiriyle işbirliği yaparak ne zaman block'ladıklarını
birbirine haber ettiği modele async programlama diyoruz. Böyle programlara da
eşzamanlı (concurrent) çalışan programlar diyoruz. Eşzamanlı çalışan bir
program herhangi anda **sadece bir iş** yapar. Sadece async bir program yazarak
paralel iki işi yürütemezsiniz.

## Blocking ne?

Eğer bir fonksiyon ne zaman block'ladığını haberdar edemiyorsa bu fonksiyon
blocking bir fonksiyondur. Bu tarz fonksiyonları hiç sevmeyiz zira kendileri
diğer haber vermeye çalışan fonksiyonları bekleterek haber vermelerini
engeller. Eğer async bir programda bilinçsiz bir şekilde blocking fonksiyonları
çağırırsanız günün sonunda daha kötü bir sync program yazmış olursunuz.

Şimdi aşağıda birkaç tane blocking kod örneği göstereceğim. Bu örneklere bakıp
blocking'in nerede olduğu tahmin etmeye çalışabilirsiniz.

**Örnek 1:**

```python
async def main(service, data):
    print("Sending data to service....")
    await service.send(data)
```

Bu örnekte `print()` blocking bir fonksiyon olduğu için bu arkadaş da
blokluyor. Şimdi diyeceksiniz ki, yahu `print()` block'lasa ne olacak, onun da
hesabını yapmayıver. Eğer programınız yüksek throughput'a ihtiyaç duyuyorsa bu
gerçekten fark ediyor.

Benzer bir sebeple `logging` statement'leri de blocking, ki logger'lar stdout'a
yazmanın yanında dosyaya yazmaya ya da mail atmaya da configure edilebiliyor
dolayısıyla bu pek şaşırtıcı olmamalı.

**Örnek 2:**

```python
from fastapi import FastAPI


def fib(n):
    if n <= 1:
        return n
    return fib(n - 1) + fib(n - 2)


app = FastAPI()


@app.get("/fib")
async def echo_number(n: int):
    return {"fib": fib(n)}
```

Bu örnekte `fib()` fonksiyonu blocking olduğu için server thread'ini meşgul
edecek. Bu fonksiyon aynı zamanda cpu-bound, bu tarz fonksiyonlar kaçınılmaz
şekilde blocking olur ve buları async bağlamında çalıştırmak istemeyiz.

**Örnek 3:**

```python
async def send_invoices_to_service(conn, service):
    coros = []
    items = await conn.execute(
        "SELECT * FROM invoices ORDER BY created LIMIT 10000;"
    )
    for item in items:
        coros.append(service.send(item))
    await asyncio.gather(*coros)
```

Burada da `for item in items` zaman zaman blocking zira potansiyel olarak on
bin tane iterasyon yapma ihtimali var, bu da Python'un yavaş for loop'ları sağ
olsun, blocking.

Genel olarak fazla iterasyonlu for loop'lar blocking olmanın yanında memory'i
inefektif kullandığınıza delalet olduğundan ya async bir batch mekanizması
varsa
onu kullanmalısınız örneğin `async for item in conn.cursor()` gibi ya da bu
örnekte ana thread'e başka bir iş yapabilmesi için zaman vermelisiniz. Mesela
her 2000 iterasyonda `asyncio.sleep` yaparak.

**Örnek 4:**

```python
import httpx
from redis.connection import cache


async def save_result_to_cache(url):
    response = await httpx.get(url)
    if response.ok:
        cache.set("result", response.content, timeout=60)
```

Burada `cache.set` blocking zira o da bir network request'i yapacak. Her client
kütüphanesi fonksiyonlarının async varyantları vermiyor veya veriyorsa bile
bunu tamamen başka bir interface ile yapıyor (bazen tamamen farklı kütüphane,
veya bazen bir hiç), dolayısıyla async varyantını bulamadığınız her şeyin sync
halini çağırmayın. Bu tarz durumlarda block etmeden çağırmanın bir yolunu
yazının devamında söylüyorum.

## Önüne `async` yapıştırınca o fonksiyon non-blocking olmaz

Örneğin yukarıdaki Fibonacci API örneğinde, blocking `fib()` async de
olabilirdi:

```python
from fastapi import FastAPI


async def fib(n):
    if n <= 1:
        return n
    return fib(n - 1) + fib(n - 2)


app = FastAPI()


@app.get("/fib")
async def echo_number(n: int):
    return {"fib": await fib(n)}
```

Bunu yapmak hiçbir şeyi değiştirmezdi zira fonksiyon ana thread'e haber
vermeyip bloklamaya devam ediyor. Dolayısıyla kullanacağınız fonksiyon herhangi
bir mekanizmayla haber veremiyorsa, onun async olmasının bir manası yok.

Bu fonksiyon için şöyle bir haber verme mekanizması kurulabilir:

```python
async def fib(n):
    if n <= 1:
        return n
    if n % 10 == 0:
        await asyncio.sleep(0.1)
    return await fib(n - 1) + await fib(n - 2)
```

Bu sayede `fib` fonksiyonu her 10'a bölünebilen bir hesaplamaya denk
geldiğinde "ya ben bir durayım, ana thread'da bekleyen elemanlar varsa onlara
`0.1` saniye zaman vereyim de onlar da bir şeyler yapsın" diyebiliyor.

> Python'da yazılmış recursive fibonacci calculator'u değil async, kimse
> kurtaramaz. Bu sadece bir örnekten ibaret. Ki burada haber verme
> mekanizması oldukça "gelişigüzel" olduğundan, daha kötü bir performans alma
> ihtimalimiz olası.

## Eşzamanlı programlar birbirleriyle paralel çalışabilir

Belki `asyncio.gather` çalışırken hiç öyle gelmese de, async bir program
paralel olarak birden fazla iş çalıştırmaz. Bu demek değil ki eşzamanlı
programlar birbirleriyle paralel olarak çalışamaz. Bunu web server'lar sıkça
yapıyor.

Örneğin uvicorn'a `--workers` verip birden fazla async runtime çalıştırmasını
sağlayabiliriz. Paralel çalıştırmayı hem thread'lar üzerinden hem de process'
ler üzerinden yapabiliriz.

Genelde Python web server'lerinde process başına bir tane async runtime
çalışır. Async runtime sayısını thread'lar kullanarak suni bir şekilde
arttırabilirsiniz fakat bu size pek bir yarar sağlamaz. Zira *teorik* olarak
sadece 1 tane async runtime kullanarak milyonlarca request serve edebilirsiniz.

Buradaki amaç CPU'yu Python'a kullandırabilmek, zira GIL yüzünden bir async
runtime CPU'yu düzgün utilize edemez. Bunu sağlamak için Birden fazla Python
process'i spawn etmemiz gerekiyor; genelde cpu sayısı kadar worker. Burada bir
nevi horizontal scaling yapmış oluyorsunuz.

## Eşzamanlı programlar thread'leri ve process'leri kullanabilir

Zaman zaman blocking bir call'i async tasarlanmış bir programın içerisinde
yapmamız gerekir zira o call'in async versiyonu yoktur. Örneğin database'den
veri çekmemiz gerekir ama kullandığımız client async desteklemiyor olabilir. Bu
durumda block etmemesi için o call'i bir ayrı thread içerisinde (async
runtime'ın dışında) çalıştırabiliriz.

Söz konusu thread görevini tamamladığında, ana thread'a haber vererek sonucu
almasını sağlar, bu sayede async olmayan bir fonksiyonu async bir programda
bloklamadan çağırabiliriz.

Örneğin bu mekanizma Django'da `sync_to_async`, FastAPI, starlette gibi
frameworklar'de ise `run_in_threadpool` wrapper'leri ile sağlanıyor. Bunlar
aslında `ThreadPoolExecutor` kullanıyor.

Bu mekanizmayı kullanmanın dezavantajları var:

1. Thread havuzu açıp kapatmanın ve maintain etmenin bir maliyeti var. Bu düşük
   bir maliyet de olsa çok fazla çağrı yapmak tüm runtime'ı etkilediği için
   yüksek istek olan senaryolarda fark ediyor.
2. Oluşturabileceğiniz thread sayısı sınırlı olduğundan, açık thread sayısı
   biriktiçe thread'leri bekleme süresi artıyor.

Async runtime'ı kullanmadaki temel amacımız zaten thread'ların
dezavantajlarından (resource kullanımı ve performans bakımından) kaçınmak,
dolayısıyla bu mekanizmayı overuse ettiğimiz zaman kazanımlarımızı egale etmiş
oluyoruz.

Öte yandan GIL yüzünden (Python interpreter'i aynı anda sadece 1 thread
çalıştırabilir) bu thread'lerin içinde de yine CPU-bound blocking task'lar
çalıştıramıyoruz!

Bir database client'in istek atması için thread oluşturmak sıkıntı değil, zira
bir network request'i yapacak; bu durumda thread response'i beklerken
Python interpreter'i salabiliyor. Fakat CPU-bound bir task için bu mümkün
değil, dolayısıyla CPU-bound bir task async bağlamında çalışmak durumundaysa,
yeni bir process oluşturup onun içinde çalıştırılmalı, bu
da `ProcessPoolExecutor` yoluyla yapılabilir.

> Database client'in istek atması için thread oluşturmak blocking değil fakat
> başka bir problemi var. Her thread bir connection paylaşmadığı için her biri
> yeni bir connection açacak ve işini bitirene kadar salmayacak. Bu da çok
> çabuk bir şekilde connection limitine takılacağınız anlamına geliyor. Bunu
> engellemek için connection pooling yapmanız gerekecek.
>
> Benzer durumları async runtime'da bulunan global bir resource'i thread'lere
> paylaştırmaya çalışırken yaşayabilirsiniz.

Yeni bir process spawn etmenin maliyeti thread oluşturmaya nispeten çok daha
fazla, o yüzden CPU-bound tasklarınızı bunlar için özel oluşturulmuş servislere
offload etmenize fayda var.

Örneği fibonacci hesaplamak için bir servis oluşturup HTTP yoluyla bu servisle
haberleşebilirsiniz, bu sayede fibonacci sayısını hesaplamak async programınız
için IO-bound bir işleme dönmüş olur. Öte yandan fibonacci servisinde Python
yerine native bir language kullanarak HTTP request maliyetini de aradan
çıkarabilirsiniz.

## Async fonksiyonlarınıza eşzamanlı çalışmalarını söylemeniz gerek

Aşağıda baştan aşağı yazılmış bir Python kodu var, bu kodu inceleyin:

```python
import asyncio
import httpx


async def main():
    r1 = await httpx.get("https://google.com")
    r2 = await httpx.get("https://microsoft.com")
    return r1, r2


if __name__ == "__main__":
    google, microsoft = asyncio.run(main())
    print(google.status_code, microsoft.status_code)
```

Yukarıdaki program her ne kadar async request kütüphanesi kullanılmış ve async
loop kullanarak çalıştırılmış olsa da sync bir program üzerine hiçbir kazanımı
yok. Çünkü yapılan işler eşzamanlı çalıştırılmamış, birinin yapılması için
diğeri beklenmiş.

Bu işlerin eşzamanlı çalıştırılması için `asyncio.gather`,
`asyncio.create_task` veya `asyncio.TaskGroup` gibi mekanizmaların kullanılması
gerekiyordu.

Şimdi bu örnekten yola çıkarak art arta çalıştırılan `await` statement'leri öcü
olarak görebilir, ve bunları tamamen yok etmeye çalışabilirsiniz. Bu da doğru
bir yaklaşım değil, çünkü işlerin ne zaman eşzamanlı çalışması gerektiğini
düşünerek yaptığınız bir seçim üzerine belirlemelisiniz.

Örneğin bir işin yapılması için başka bir işin sonucu gerekiyorsa burada
zaten `await` zincirlemek zorundasınız. Öte yandan, istek atacağınız servis'in
karşılayabileceği istek sayısı sınırlı ise (ki hep öyledir), ancak belirli
sayıda fonksiyonu eşzamanlı olarak çalıştırabilirsiniz.
Unutmamanız gereken önemli bir nokta: her `await` statement'i aslında *"ben
artık bu fonksiyon'a bir mola veriyorum ve event loop'a bekleyen başka
fonksiyonları çalıştırabileceğini veya molası biten diğer fonksiyonlar varsa
onlara devam etmesini gerektiğini söylüyorum"* demek.

Bu örnekte molada olan veya başlamayı bekleyen başka bir fonksiyon olmadığı
için await statement'i eşzamanlılık bağlamında hiçbir işe yaramıyor. Fakat bu
kodun aynısı alıp bir API endpoint'i yapsaydık bu sefer eşzamanlı olarak birden
fazla istek alıp bunları cevaplayabilirdik, her ne kadar iki HTTP request'i
yine senkron çalışacak olsa da.

Ek olarak bu spesifik program sync haline nispeten daha yavaş çalışıyor, çünkü
eşzamanlı olmayışının yanında, programın başlarken bir async runtime oluşturma
overhead'i var. Bu da *özünde* async programların sync programlardan daha hızlı
olmadığını bize gösteriyor; bu sadece farklı bir programlama tekniği.
