+++
title = "Python ile metaprogramlamaya bir bakış"
date = 2022-02-13T14:26:26.690Z
tags = ["python", "turkish"]
slug = "python-metaprogramlama-giris"
+++

Haddim olmasa da bu konu hakkında bir yazı yazmak istiyorum zira bu konu
hakkında Türkçe kaynakların bolluğundan yakınmıyoruz; aynı zamanda yazmak
kişisel öğrenmemin de bir parçası.

Önce tanımlamak gerekirse, metaprogramlama (programlama üstü ya da üst
programlama) adından da anlaşılacağı üzerinde program programlamayı bir problem
olarak inceliyor.

Bir başka deyişle, normal işlerimizde bir program yazarız ve bu program bir
“problem”i çözer. Eğer çözdüğünüz “problem” programlama ile ilgili ise
metaprogramlama yapıyorsunuz demektir.

Metaprogramlama yapılarını herkes kullanabilir ama ekseriyetle SDK/API ve
kütüphane tasarımcılarının işine yarıyor zira kullanıcılara çok fazla kod
yazdırmadan zengin bir işlevsellik sunmuş oluyorlar.

## Decorator’lar

Neredeyse her Python programcısı eninde sonunda bu yapıyı kullanır, o yüzden bu
yapının nasıl çalıştığı konusunda bir içgörü edinmek oldukça önemli. Eğer
decorator nedir bilmiyorsanız, ya da kafanızda canlanmadıysa şu örneğe
bakabilirsiniz:

```python
@app.route('/')  
def home():  
    return render_template('home.html')
```

Bu örnek Flask’dan, `app.route` isimli bir decorator, `home` isimli bir
fonksiyona eklenmiş. Sadece bu basit eklentiyle, sıradan bir fonksiyon
controller’e çevrilmiş oluyor, işte fazla kod yazdırmadan işlevsellik
kazandırmanın en basit örneklerinden biri.

Peki decorator’lar nedir? Biliyorsunuz ki Python’da **her şey** bir nesne,
mesela  `1.5 + 2.5`  yazmak yerine `float` nesnesinin metodunu
`1.5.__add__(2.5)` şeklinde çağırabiliyoruz.

Bazı nesneler var ki bunların çağrılabilme özellikleri var, örneğin
fonksiyonlar veya metotlar; bunları çağırmak için de `()` kullanıyoruz. `int`
de bir nesne bildiğiniz gibi, eğer bu nesneyi çağırırsanız size 0 döner
(bir argüman verirseniz de o argümanı int’e cast eder/çevirir ve onu döner).
Fakat  `5`  ‘i düşünelim, bu da bir nesne ve tipi `int` ama bu nesne
çağrılabilir değil:

```python-repl
>>> int()  
0  
>>> 5()  
<stdin>:1: SyntaxWarning: 'int' object is not callable; perhaps you missed a comma?  
Traceback (most recent call last):  
File "<stdin>", line 1, in <module>  
TypeError: 'int' object is not callable
```

Bu bilgilerle birlikte, en ilkel tabiriyle, bir decorator argüman olarak bir
**çağrılabilir** alan bir  **çağrılabilendir**.

_Hatta çoğu decorator çağrılabilen döner._

> “Yahu Şuayip, sen int() deyince, aslında `int.__init__` ya da `int.__new__` 
> çalışıyor, yani sen çağırma yapmıyorsun, aslında class initialize ediyorsun!”
> diyebilirsiniz. Çağırabilme kavramı biraz daha karmaşık olabilir, o yüzden
> bir  [C implementation’a](https://stackoverflow.com/a/115349/6063532)
> bakın, emin olmak için Python’da builtin `callable()` ile bir nesenin
> çağırabilir olup olmadığını öğrenebilirsiniz.

Çağrılabilme kavramını anladıysanız oldukça basit bir tanımı ve yapısı var
decorator’ların. Şimdi örnek bir decorator görelim ve açıklayalım:

```python
def deco(func):  
    print("Decorated %s" % func)      
    return func
```

Bu örnek decorator, decorate ettiği çağrılabilir nesneyi her decorate
ettiğimizde decorate ettiğimiz şeyi yazdırıyor. Tabii bu pek yararlı değil zira
genelde biz nesnenin çağırdığı senaryodaki işlevselliğini değiştirmek isteriz.

Yine de bazen programın ilk çalışma anında  decorate edilen nesnelerin
davranışını değiştirmek için bu yapıyı kullanabiliriz. Mesela bir class’i
decorate ettiğimizi düşünelim, bu yapıyı kullanarak class’ın kendisine ait bazı
metotları ve attribute’ları değiştirebiliriz. Ya da daha uçuk kaçık bir şey
yapıp, decorate ettiğimiz class’i döndürmek yerine başka bir class veya nesne
döndürebiliriz (bunu bir nevi sonraki adımda yapıyoruz).

Şimdi nesnelerin çağırılma anına etki eden bir decorator yazalım:

```python
def deco(func):  
    def inner(*args, **kwargs):  
        print("Calling %s" % func)  
        return func(*args, **kwargs)  
  
    return inner
```

Gördüğünüz üzere çağrılma anına etki etmek için bir fonksiyon tanımı yapmamız
gerekiyor. İşin aslında, herhangi bir nesne alıp yeni, alakasız bir function
nesnesi geri döndürüyoruz. Bu yeni function nesnesi de scope’unda bulunan, yani
decorate edilmiş nesneyi çağırıyor, böylelikle orijinal nesneyi taklit ediyor.

Bu yapıyı, fonksiyon bağlamında düşünürsek, fonksiyonu her çağırdığımızda ek
olarak bir şeyler yazdırıyoruz. Bu senaryo genelde log amaçlarıyla
kullanılabiliyor; mesela fonksiyonlar ne zaman çağrılmış kaydı tutulsun diye.

Şimdi decoratorlar’ların nasıl uygulandığı konusunda ufak bir parantez açalım.
Yani decorator yazdık iyi tamam, peki bir fonksiyonu nasıl decorate edeceğiz?
Şimdi örneği görelim:

```python
def echo(number):  
    return number  
  
echo = deco(echo)
```

Decorator’lar bahsettiğimiz üzere çağrılabilir alan çağrılabilen olduklarından,
decorate etmek istediğimiz nesneyi argüman olarak decorator’a vermemiz yeterli.
Tabii bu sırada decorate ettiğimiz nesneyi yeniden tanımlamamız gerekiyor (ki
aynı isimle kullanabilelim).

Decorator’ların uygulanma prensibinin temelde bu olduğunu anlamak önemli zira
sıradaki göstereceğim syntax’i direk yutarsanız ne yaptığımızı anlamanız
zorlaşabilir. İşte decorator’ları uygulamanın bir başka yolu:

```python
@deco  
def echo(number):  
    return number
```

Bu syntax tamamen işimizi kolaylaştırmak, sürekli tekrar eden kod yazmamak için
Python tarafından bize sunuluyor; çoğunlukla da decorator’lar böyle uygulanır.
[Bu syntax’in](https://en.wikipedia.org/wiki/Syntactic_sugar) temelde yaptığı
şeyin yukarıdaki ile aynı olduğunu tekrar hatırlatmak istiyorum.

Şimdi şu koda bir bakalım:

```python
def deco0(func):  
    print("Decorating %s" % func)  
    return func  
  

def deco(func):  
    def inner(*args, **kwargs):  
        print("Calling %s" % func)  
        return func(*args, **kwargs)  
  
    return inner  


def echo(number):  
    return number  
  
  
echo_1 = deco(echo)  
echo_2 = deco0(echo)  
  
echo_1 is echo  # output: False  
echo_2 is echo  # output: True
```

Yukarıda yine örnekler üzerinden 2 tane decorator tanımladım, bir fonksiyonu da
bu ikisi decorator ile decorate ettim. deco0'da direkt nesnenin kendisini,
deco’da ise yeni oluşturduğum fonksiyonu dönüyorum.

Son iki satırdaki statement’lerin çıktılarını incelersek, yeni fonksiyon yani
yeni bir nesne dönen decorator’un decorate ettiği fonksiyonu değiştirdiğini
görüyoruz. Decorate ettiğimiz fonksiyon çağrıldığında yine eski nesnenin
davranışı taklit edileceği için bu durumda sorun olmayacaktır, yani girdiğimiz
sayı aynen bize dönecektir, beklediğimiz gibi.

Peki ya çağrılmadan önceki durum nedir? Decorate edilmiş nesne kendisi gibi
değil, içerde döndüğümüz inner fonksiyonu gibi davranıyor! Bunu tabii ki
istemeyiz. Bunun yol açtığı bazı sıkıntılar var, mesela decorate ettiğiniz
function’a docstring koymuşsanız bu kaybolur. Ya da daha dramatik olarak, inner
metodu orijinal nesnenin imzasıyla aynı değilse metodun imzasını kaybedersiniz.
Örneğin bu bağlamda decorate edilmiş fonksiyonu `echo(254, domates=123)`
şeklinde çağırmanızda problem olmaz (ancak bu argümanlar orijinal metoda
gidince hata çıkar, o sürece kadar giden tüm kodlar gereksiz çalışmış olur).

Bu durumu düzeltmek için bir aracımız var, ne tesadüf ki bu araç da bir 
decorator. Şimdi bu decorator’un düzgün haline bakalım:

```python
import functools

def deco(func):
    @functools.wraps(func)
    def inner(*args, **kwargs):
        print("Calling %s" % func)
        return func(*args, **kwargs)

    return inner
```

yani:

```python
def deco(func):  
    def inner(*args, **kwargs):  
        print("Calling %s" % func)  
        return func(*args, **kwargs)  
  
    return functools.wraps(func)(inner)
```

`functools.wraps` decorate ettiği fonksiyonu, argüman olarak veren fonksiyona
benzetmeye yarayan bir araç. Burada benzetme kelimesi önemli, zira decorate
ettiğiniz nesne her zaman değişiyor; önemli olan bu nesnenin davranışının 
olabildiğince aynı kalması. wraps, yukarıda bahsettiğim problemleri ve
birkaçını daha çözüyor; bu metodu nesneyi değiştiren her decorator yazdığımızda
kullanmalıyız.

Daha önceden decorator’ların çağrılabilen alan bir çağrılabilir olduğunu
söylemiştim. Dikkat ederseniz, wraps daha tanımında çağrılabilen alıyor,
üzerine de bir function decorate ediyor. O zaman wraps decorator döndüren bir
decorator mü oluyor? Muhtemelen hayır; bunlara argüman alabilen decorator’lar
diyoruz. Ama “decorator döndüren decorator” gayet geçerli ve istek ve eminim ki
örnekleri vardır.

Bu aşamada argüman alabilen decorator nasıl yazılır’ı anlatmayacağım, ama
“decorator alan decorator” düşüncesi size fikir vermiş olmalı. Hemen küçük bir
parantezde, [argüman alan decorator yazmayı kolaylaştıran kütüphanemin](https://github.com/realsuayip/decfunc)
reklamını yapmak isterim; bunu hem de class yazarak yapıyorsunuz!

> Çağırma anına etki eden bir decorator’un, illa ki function dönmesine gerek
> yok, herhangi bir çağrılabilir nesne dönebilir. Fakat wraps function
> dışındaki nesneler için düzgün çalışmıyor. O yüzden genelde function
> dönülmesi hayırlı olur. Bu neyi decorate ettiğinize göre değişir tabii.

## type

Eğer bir nesnenin type’ini merak edersek bunu tek argüman olarak bu arkadaşa
verebiliriz; bunu hepimiz biliyoruz.

Daha az bilinen şey ise type, 3 argüman aldığı zaman yeni bir ‘type’ oluşturur.
Peki ‘type’ nedir? Önce bir class tanımı yapalım:

```python
class Person:  
    def __init__(self, name):  
        self.name = name  
  
ismet = Person(name="İsmet")
```

Bu bağlamda `type(ismet)` ne diye bakarsanız `Person` olduğunu görürsünüz. Peki
bir de `type(Person)` neymiş diye bakarsak onun da `type` olduğunu görüyoruz.

Sonuç olarak bir bir class tanımı yaptığımız zaman Python gidip bize özel bir
type nesnesi oluşturuyor (bu bağlamda `Person`). Python’da her şey bir nesne
demiştik; **class’lar de bir nesnedir ve `type`'ın subclass’ı olurlar.**

> Oysa type’in type’ina bakarsaniz yine type karşınıza çıkar. Demek ki
> gökküşağının sonunda type var.

Bir type oluşturulduğunda birkaç karakteristik özelliği olur:

1.  İsmi ne?
2.  Hangi base classları almış, yani neyi inherit etmiş?
3.  Class attribute’leri neler, yani class gövdesinde hangi tanımlar yapılmış?

Ancak bu üç bilgi elimizde ise, bir type oluşturabiliriz. Peki nasıl ya da ne
kullanarak type oluşturabiliriz? Metaclass kullanarak.

Bir class’ın davranışını belirleyen classlara ‘metaclass’ denir; bu
davranışlardan biri de oluşturulma şeklidir. Eğer bir metaclass belirtmezseniz,
oluşturduğunuz tüm type’lar ‘type’ metaclass’i ile oluşturulur, yani şu şekilde:

```python
def __init__(self, name):  
    self.name = name  
  
Person = type("Person", (), {"__init__": __init__})  
ismet = Person(name="İsmet")
```

Buradaki argüman’lar yukarıda belirttiğim karakteristik özelliklere denk
geliyorlar. Sonuç olarak anlamamız gereken şunlar:

1.  Her class aslında type’ın bir subtype’ıdır.
2.  Type’ları oluşturmanın tek yolu, class syntax’ı değil.
3.  Type’ların oluşturulmasını (ve diğer davranışlarını) değiştiren veya
kontrol eden özel type’lar var.

## Metaclass kavramı

Metaclass’ların ne olduğunu yukarıda öğrenmiştik. Şimdi kendimizi metaclass
yazmak için motive etmeye çalışalım. Mesela bir class’ın oluşum sürecine neden
müdahale etmek isteyelim ki?

Bunun üzerine çokça düşünürseniz, bir sebep bulamayacaksınız zira metaclass
kullanımı günlük programlama rutinimiz için çok ileri bir çözüm. Genel tavsiye
de, kodun karmaşıklığını çoğaltmamak için metaclass yazılmaması yönünde.

Şimdi sizin için bir metaclass kullanma bahanesi üreteceğim. Diyelim ki
yukarıda öğrendiğimiz decorator bilgileriyle çok güzel bir class decorator’u yazdık (bu decorator’un ne yaptığı önemli değil). Fakat bu decorator’u gidip 50 tane class’in üstüne decorate etmemiz gerektiğini fark ettik, tabii canımız çok sıkıldı; çok fazla kod tekrarı yapacağız. İleride bu decorator’un bir argümanınını değiştirmek istersek vay halimize!

Metaclass bu bağlamda bir çözüm sağlayabilir. Bu classların base classına özel
bir metaclass uydurup, classın oluşturulma aşamasında class’ı decorate
edebiliriz. **Metaclass subclasslara propagete ettiği için** tüm subclasslar
da decorate olmuş olur. Şimdi örnek üzerinden inceleyelim:

```python
class PersonMeta(type):  
    def __new__(mcs, name, bases, classdict):  
        cls = super().__new__(mcs, name, bases, classdict)  
        cls = super_useful_decorator(cls)  
        return cls  
  
  
class Person(metaclass=PersonMeta):  
    def __init__(self, name):  
        self.name = name
```


Şimdi buradaki anahtar gözlemleri sıralayalım:

1.  Metaclass type’in subclass’ı, zira type da bir metaclass. Metaclass 
işlevselliği için bu subtyping’a ihtiyacımız var (zira arkada C ile dönen bir
implementation var; oraya kadar inemiyoruz).
2.  Buradaki `__new__` sıradan `__new__` ile aynı imzaya sahip değil, normalde
`__new__`, `__init__`’den önce çalışır ve **instance’yi oluşturmakla**
görevlidir. Oysa burda **class’ın kendisini oluşturmakla** görevli.
3.  `__new__`’in imzası `type()`’nın 3 argüman alan imzasıyla aynı, yani
yukarıda bahsettiğimiz üç karakteristik.
4.  Bir class’ın metaclass’ını belirtmek için, class tanımında metaclass
keyword argümanı kullanılır.
5.  `__new__` içindeki oluşturma mekanizması için yine type’a başvuruyoruz
(super yoluyla; burada açık açık `cls = type(name, bases, classdict)` da
diyebilirdik, ama yaygın kullanım super’i çağırmaktır). Demek ki type
oluşumuna müdahele etmek aslında o kadar karmaşık bir işlem değil. Ya bu üç
karakteristiği değiştireceğiz ya da yeni oluşturulan type nesnesi üzerinde
birtakım işlemler yapacağız.

Yani görünen o ki yukarıdaki verdiğim basit template ile `__new__` gövdesinde
birtakım özelleştirmeler ile type oluşumunu değiştiren bir metaclass
yazabiliyoruz. Tabii sadece `__new__` metodunu değiştirmek zorunda değilsiniz,
ama en çok değiştirilmek istenen metot bu. Diğer metotların çalışma
mekanizmaları için örn. `__call__` daha incelikli detaylar var, bunları
kullanmadan önce araştırmalısınız.

Tabii Python developer’ler de metaclass’ların pek karmaşık yapılalar
olduklarını anlamışlar ki bunların üzerine bazı soyutlama katmanları yapmışlar.
Bunlardan iki tanesi şunlar: şimdi bahsedeceğim `__init_subclass__` ve daha
sonra descriptor kısmında bahsedeceğim `__set_name__` metotları.

Metaclass’a başvurmanın en sık sebebi bir ‘kayıt etme’ işlevselliği oluşturmak.
Django üzerinden bir örnek vereyim. Django’da model kavramı var ve bunlar
özetle class tanımlarını database tablolarına çeviriyor. Bu aşamada Django 
“acaba hangi class’lar model?” diye merak ediyor zira database tarafında
tablolarını oluşturacak. Bu yüzden her Model classını inherit alan bir class
oluşturduğumuzda Django bunu kayıt edecek bir mantığa ihtiyaç duyuyor.

Bu aşamada demin aynı decorator’lar için uyguladığımız metodu uygulayıp, bu 
sefer decorate etmek yerine classları bir listeye atan bir logic yazabilirdi.
Bu sayede Model class’ini subtype etmiş tüm classların listesi elimizde olurdu.

`__init_subclass__` metodu da tanımlandığı class’tan başka bir class 
**inherit edince** çalışan bir metot ve yukarıdaki senaryo için bire bir. Bir
örnek vermem gerekirse:

```python
registry = []  


class Person:  
    def __init__(self, name):  
        self.name = name  
  
    def __init_subclass__(cls, **kwargs):  
        super().__init_subclass__(**kwargs)  
        registry.append(cls)
```

Bu Person class’inden inherit eden herhangi bir class oluşturduğumuz zaman,
registry’nin dolduğunu göreceksiniz. Aynı zamanda bu metot için gelişigüzel
keyword argümanları da tanımayabilirsiniz, bu argümanlar class’ın tanımında
verilebiliyor, mesela:

```python
class Person:  
    def __init__(self, name):  
        self.name = name  
  
    def __init_subclass__(cls, location, **kwargs):  
        super().__init_subclass__(**kwargs)  
        cls.location = location  
        registry.append(cls)  
  
  
class RichPerson(Person, location="Los Angeles"):  
    pass
```

Tabii bu özelliği metaclass `__new__` de sağlıyor (classdict sonrası keyword
argument ekleme yoluyla). Aynı zamanda `__init_subclass__` kullanmak için
herhangi bir metaclass tanımı yapmadığımıza dikkat edin.

Son olarak, metaclass yazarken farkında olmanız gereken bir şey,
**yazdığınız metaclass davranışını değiştireceği class’dan bağımsız, kendine
münhasır bir class**. Yani metaclass’ın gövdesine bir metot koyayım ve
oluşturduğum class’lar bu metodu çağırabilsin diye bir mantık yok; ki böyle bir
mantığa gerek de yok, direkt oluşturduğunuz class’a metot enjekte edebilirsiniz
(ya da düz inheritance kullanabilirsiniz). Bunu standart inheritance’ye
alıştıysanız kafanızı karıştırabilir diye özellikle söylüyorum.

## Descriptor’lar

Descriptor’lar kısaca class’ların attribute erişimini özelleştirmemize yarıyor,
bir başka tabirle “nokta operatörünü özelleştirmek”. Mesela şu örnek üzerinden
gidelim:

```python
class Person:  
    name = None  
  
person = Person()  
person.name = "İsmet"
```

Bu örnek güzel ama mesela şunları yapmak istesek nasıl yapardık acaba?

1.  Eğer Person’a name atanacak ise, bu name en az 3 karakter olsun, değilse
hata versin.
2.  Eğer Person’un name’si belirlenmemiş ise, person.name diye çağırdığımız 
zaman “isim belirilmemiş” diye hata versin.
3.  Person’a name atanınca Person’un eski isimleri bir listede tutulsun ve
person.old_names şeklinde erişebilelim.

Şimdi biraz beyin jimnastiği yaparsanız ve kendinizi çok kasarsanız bunların
işlevselliğin hepsini oldukça karmaşık metaclass yapısıyla bir şekilde
halledebileceğimizin farkına varabilirsiniz. Tabii bunu istemeyiz, bunun yerine
descriptor’un sağladığı soyutlaştırmayı kullanabiliriz. Şimdi yukarıdaki
işlevselliğin kazandırıldığı bir örnek görelim:

```python
class Name:  
    names = []  
    current_name = None  
  
    def __set__(self, instance, value):  
        if len(value) < 3:  
            raise ValueError(  
                "Please provide a name that"  
                " is at least 3 characters."  
            )  
  
        self.names.append(value)  
        instance.old_names = self.names[:-1]  
        self.current_name = value  
  
    def __get__(self, instance, owner):  
        if self.current_name is None:  
            raise ValueError("Name is not set.")  
  
        return self.current_name  
  
  
class Person:  
    name = Name()
```

İlk bakışta biraz karmaşık gözüküyor olabilir, ama basit bir mantıkla
kurulduğunu `__set__` ve `__get__` metotlarını sindirdikten sonra
anlayabiliriz. Şimdi birkaç çıkarım/tanımlama yapalım:

1.  Descriptor’lar class gövdesinde initialize edilir. Her descriptor kendi
başına bir class’dır.
2.  `__set__` metodu attribute değişiminde çağrılır, yani özetle erişimi bir
metot ile sarmış oluyoruz. Benzer bir şekilde bu durum `__get__` için de
geçerli, bu metot da erişim zamanında çağrılır.
3.  Eğer bir class `__get__`, `__set__` ya da `__delete__` tanımı yapıyorsa,
bir descriptor olur.

Sonuç olarak descriptor’lar “nokta” ile yapılan işlemleri bir metoda sararak
dinamik erişim ve değişim özellikleri sunuyor. Bu metotların kullanımını
değerlendirerek çeşitli işlevselliğe ulaşabiliriz. Bu bağlamda validation
işlemleri için kullandık mesela.

_`__delete__` metodu `del person.name` için bir özelleştirme sunuyor, fakat
bunun üzerinde fazla durmayacağım._

Eğer önceden bir framework’ta buna benzer bir yapı kullandıysanız, syntax’i
size tanıdık gelmiş olabilir. Örneğin Django’da ve pek çok ORM’da şu yapı
vardır:

```python
class Person(models.Model):  
    name = models.CharField()
```

Peki Django bu yapıyla ne yapıyor? `__set__` ve `__get__` metotlarını
kullanarak ilgili bilgiyi arkada sırasıyla database’e yazıyor veya çekiyor.

Şimdi aklımıza gelmesi gereken bir soru, Django database’da tabloya yazarken
veya bilgiyi çekerken kolon ismini nasıl biliyor? Yani şu query’i yapabilmek
için “name”nin bir yerden gelmesi lazım:

```sql
SELECT name FROM users WHERE user.id = 1
```

Burada da imdadımıza `__set_name__` yetişiyor. Bu metot bize descriptor’un
nasıl isimlendirildiği konusunda bilgi veriyor. Örnek verelim:

```python
class Name:  
    names = []  
    current_name = None  
  
    def __set_name__(self, owner, name):  
        self.field_name = name  
      
    ...
```

Eğer bundan sonra, descriptor ne diye isimlendirilmiş merak edersek
`self.field_name` kullanabiliriz. Bizim bağlamımızda bunun değeri `name`
olacaktır.

_Önceden bahsettiğim üzere `__set_name__` de metaclass üzerine bir
soyutlaştırma, ve metaclass’a nispeten yeni bir özellik._

Descriptor konusunda ufkunuzun açılması için Python dokümantasyonundan bir
örnek vereceğim. Buradaki descriptor verilen dizindeki dosya sayısını dinamik
olarak almaya yarıyor:

```python
class DirectorySize:  
    def __get__(self, instance, owner):  
        return len(os.listdir(instance.dirname))  
  
class Directory:  
    size = DirectorySize()  
  
    def __init__(self, dirname):  
        self.dirname = dirname
```

Tabii bu yapıyı daha basit bir şekilde class içinde de halledebilirdik, ama
fikrimce bu örnek descriptor’ların dinamik yapısını iyi bir şekilde gösteriyor.

Descriptor’ların da `__init__` metotları olduğundan, tanımlarken çeşitli
argümanlar da verebiliyoruz. Mesela Django `CharField` için `max_length` gibi
bir argüman alabiliyor bu da database bağlamında `VARCHAR` boyutunu belirliyor.

Sonuç olarak descriptor’lardan anlamamız gereken şunlar:

1.  Bir class’ın attribute erişimini ve değişimini kontrol eden mekanizmalara
descriptor diyoruz ve bu işlemler sırayla `__get__` ve `__set__` metotlarına
denk geliyor.
2.  Descriptor’lar özellikle data validation veya dinamik data erişimi için
işimize yarıyor.
3.  Descriptor’lar argüman alabiliyorlar ve `__set_name__` metoduyla bir
descriptor’un nasıl isimlendirildiğini öğrenebiliyoruz.
4.  Descriptor’lar sadece class variable olarak tanımlandıkları zaman
çalışırlar.

Son olarak Python’daki classmethod, staticmethod ve property gibi decorator’lar
de descriptor yöntemiyle çalışıyorlar. Örneğin class’daki bir metodu property
decorator’u ile decorate ettiğimiz zaman bir descriptor tanımlanmış oluyor ve
bu descriptor arkada decorate edilmiş metodu çağırıyor.

Dikkat ettiyseniz, metaprogramlama yapıları birbiriyle iç içe geçmiş durumda,
“aslında descriptor olan bir decorator” veya “metaclass yoluyla kendini
manifesto eden decorator’lar” da bunlara örnek. Demek ki metaprogramlama
yapıları birbiriyle çok ilişkili ve bir harmoni içinde kullanıldıkları zaman
oldukça işlevsel özellikler sunuyorlar. Bu bağlamda yine metaprogramlama
yapabilmek için dilin kendisini ve veri yapılarını çok iyi şekilde anlamamız
gerektiği ortaya çıkıyor.
