+++
title = "Python’da distributed lock mekanizması"
date = 2023-12-22T22:18:06.972Z
tags = ["python", "redis", "multiprocessing", "django", "turkish"]
slug = "python-distributed-lock"
+++

Bildiğiniz üzere, eşzamanlı programlamada lock bir senkronizasyon primitifidir.
Bu arkadaş sayesinde, eşzamanlı çalışan iki fonksiyonun tek bir kaynağı aynı
anda yönetmeye çalışmasını engelleyebilirsiniz; bu sayede
[race condition](https://en.wikipedia.org/wiki/Race_condition) senaryolarının
önüne geçmiş olursunuz.

Farzımuhal bir cüzdan uygulamasında, müşteri hesabından para çektiğinde
bakiyesinin azalması senaryosunu düşünelim. Bunu yapmak için kullanıcın mevcut
bakiyesinden, çekilen tutarın çıkartılıp veri tabanına kaydedilmesi gerekir.

Eğer kullanıcı aynı anda başka bir ödeme işlemi de yapıyorsa ve o işlem de
bakiyesini değiştirecekse bu işlemleri bir sıraya sokmamız gerekir, zira bakiye
sorgulama işlemi yapıldığı anda —aynı anda çalıştıklarından— ikisi de aynı
bakiyeyi göreceklerdir.

Dolayısıyla 100 lira bakiyesi olan bir müşteri, aynı anda 2 tane 25 liralık
bakiye düşürme işlemi yaparsa, ve buradaki race condition düzgün handle
edilmezse, kullancının bakiyesi 50 lira yerine 75 liraya düşer.

Her bağlamda ve domain’de farklı lock çözümleri vardır. Örneğin yukarıdaki
bağlamda, locking işlemini database seviyesinde yapabiliriz. Bunun için
mesela `SELECT ... FOR UPDATE` gibi bir mekanizma bulunur. Eğer thread’ler
ile çalışıyorsanız yine kullandığınız programlama dili, o dilde implement
edilmiş lock mekanizmalarını size sunar; mesela bu mekanizmayı kullanarak aynı
process içindeki thread’lerin birbiriyle race condition oluşturmasını
engelleyebilirsiniz.

Bu yazıda anlatacağım lock, bu farklı domain’lerden bir tanesi. Distributed
lock, yani Türkçe’de “dağıtılmış” lock, **makineler arası** lock
mekanizmaları kurmanıza yarar. Hatta bu makinelerde aynı process’lerin
çalışmasına bile gerek yok.

Örneğin Python ve JavaScript ile yazdığınız iki farklı servisiniz olsun ve bu
servislerin aynı anda çalıştırıldığı zaman race condition oluşturacakları bir
senaryo olsun. Distributed lock kullanarak, JavaScript kodunun Python kodunun
bitmesini beklemesini sağlayabilirsiniz (veya tam tersi).

Şimdi Python kullanarak naive bir distributed locking mekanizması oluşturalım.
Bunu yapmak için bir cache server’e ihtiyacınız var (örneğin Redis) zira hangi
lock’un acquire edildiğini takip etmemiz gerekiyor.

Distributed lock’a bakmadan önce çok önemli bir not:

> Genel olarak senkronizasyon ihtiyaçlarınız için açık bir şekilde lock
> **kullanmamalısınız**. Lock senkronizasyon için soyutlama seviyesi en düşük
> primitiflerden biridir ve handle etmesi oldukça zordur.
> [Deadlock](https://en.wikipedia.org/wiki/Deadlock) hatalarının debug etmesi
> oldukça zordur, hele ki dağıtık bir bağlamda.
>
> Lock yerine  [**Queue**](https://docs.python.org/3/library/queue.html)
> yapılarını kullanabilirsiniz. Yani bir kaynağı kilitlemek yerine, o kaynağa
> yapılan erişimleri sıraya koyarak tek tek işleyebilirsiniz.

Aşağıda örnek bir lock implementasyonu var, kodu okuyalım ve anlamaya
çalışalım:

```python
import time
from contextlib import contextmanager

from django.core.cache import cache


@contextmanager
def lock(lock_id: str):
    while True:
        acquired = cache.add(lock_id, 1)
        if acquired:
            try:
                yield
            finally:
                cache.delete(lock_id)
                break
        else:
            time.sleep(0.1)
```

1. İlk önce cache server’e gidiyoruz, verilen lock numarasını cache’ye
   kaydetmeye çalışıyoruz. Burada ben örnek olarak `1` sayısını kaydediyorum,
   siz isterseniz makinenin hostname’ini ekleyebilirsiniz. Bu sayede lock hangi
   makine tarafından acquire edilmiş görebilirsiniz, debug işleri kolaylaşır.
2. Buradaki `add` fonksiyonu önemli, bu fonksiyon eğer kayıt yoksa yeni kayıt
   ekliyor ve `True` dönüyor, eğer kayıt halihazırda varsa `False`dönüyor ve
   bir şey yapmıyor. Bu mekanizmayı kullanarak lock’u acquire edip etmememiz
   gerektiğini anlıyoruz. Kullanacağınız cache server’de muhtemelen bu tarz bir
   fonksiyon olacaktır.
3. Eğer lock’u acquire ettiysek, context manager gövdesine girip kodların
   çalışması için `yield` ediyoruz. Gövde çalıştıktan sonra `finally`
   kısmında lock’u serbest bırakıp döngüyü sonlandırıyoruz.
4. Eğer lock’u acquire edemezsek, yani başka biri tarafından
   kullanılıyorsa, `0.1` saniye kadar bekliyoruz ve tekrar deniyoruz. Lock
   boşa çıktığı zaman 3 numaralı adımları tekrar yapıyoruz.

Gördüğünüz üzere implementasyon’un aslında normal bir lock’dan pek bir farkı
yok. Sadece lock acquire edilmiş mi bilgisini ortak bir cache server’de
tutuyoruz.

Yukarıdaki örnekte Django’nun cache kütüphanesini kullandım, fakat siz cache
server’le iletişime geçmenizi sağlayacak herhangi bir kütüphane
kullanabilirsiniz.

Daha da önemlisi, bu kütüphaneler zaten lock mekanizmasını sizin için sofistike
bir şekilde implement ediyorlar; bu tarz bir implementasyon yerine onları
kullanmalısınız (zira örneğin burada lock timeout bile yok).

Şimdi bu implementasyon üzerinden bir örnek göstereyim, aşağıda iki tane
fonksiyonum var. Bu fonksiyonların biri uzun, diğeri kısa sürüyor. Bu
fonksiyonlar yine aynı lock’u acquire etmeye çalışacaklar:

```python
def short_process():
    then = time.monotonic()
    with lock("hello"):
        time.sleep(0.5)
    print("[Short] Elapsed: %s" % (time.monotonic() - then))


def long_process():
    then = time.monotonic()
    with lock("hello"):
        time.sleep(3)
    print("[Long] Elapsed: %s" % (time.monotonic() - then))
```

Farz edin ki, bu fonksiyonlar aynı anda çağrıldılar. Nasıl bir davranış
beklerdiniz? Yukarıdaki implementasyonu ele alarak tahmin etmeye çalışın.

Fonksiyonları aynı anda çalıştıran bir fonksiyon yazalım, ve bu fonksiyonun
sonuçlarına bakalım:

```python
def run_in_parallel():
    then = time.monotonic()
    with concurrent.futures.ThreadPoolExecutor() as executor:
        executor.submit(short_process)
        executor.submit(long_process)
    print("[Cumulative] Elapsed: %s" % (time.monotonic() - then))
```

```python-repl
In [3]: run_in_parallel()
[Short] Elapsed: 0.5056477510006516
[Long] Elapsed: 3.520465544000217
[Cumulative] Elapsed: 3.5220909180006856

In [4]: run_in_parallel()
[Long] Elapsed: 3.003538042999935
[Short] Elapsed: 3.6081660849995387
[Cumulative] Elapsed: 3.608987043000525

In [5]: run_in_parallel()
[Short] Elapsed: 0.5059752079996542
[Long] Elapsed: 3.5184125019995918
[Cumulative] Elapsed: 3.520371127000544

In [6]: run_in_parallel()
[Short] Elapsed: 0.5088362090000373
[Long] Elapsed: 3.518181292999543
[Cumulative] Elapsed: 3.520474001000366

In [7]: run_in_parallel()
[Long] Elapsed: 3.0075693770004364
[Short] Elapsed: 3.5897362099995007
[Cumulative] Elapsed: 3.590527919000124
```

Tek başlarına 3 saniye ve 0.5 saniye süren bu fonksiyonlar,  **aynı anda
çalışmaya** başlıyorlar ve kümülatif bir şekilde her zaman ~3.5 saniye
sürüyorlar. Zira yaptıkları işlemleri lock kullanarak sırayla
gerçekleştiriyorlar.

Yine bazen kısa süren fonksiyon önce lock’u acquire ediyor, bazen de uzun süren
fonksiyon.

Örneğin kısa süren fonksiyonun önce acquire ettiği örnekte, uzun fonksiyon
toplam 3.5 saniye sürmüş, buradaki fazladan 0.5 saniye, kısa fonksiyonun lock’u
boşa salmasını beklemekle geçiyor

Bunun tam tersi de doğru; kısa fonksiyon 3 saniye boyunca uzun fonksiyonun
bitmesini bekliyor.

Peki, bu mekanizmayı gerçek hayatta nerede kullanabiliriz? Diye soruyor
olabilirsiniz. Benim gördüğüm örneklerden
[birinde](https://docs.celeryq.dev/en/stable/tutorials/task-cookbook.html#ensuring-a-task-is-only-executed-one-at-a-time)
aynı anda tek bir Celery task’ının çalışması için kullanılmış mesela.

Eğer kullandığınız üçüncü parti bir servis (API, database vs.), size lock
mekanizmaları sunmuyorsa yine bu çözüme başvurabilirsiniz.
