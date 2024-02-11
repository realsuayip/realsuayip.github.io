+++
title = "iptables’e Giriş"
date = 2018-08-16
tags = ["linux", "turkish"]
slug = "iptables-giris"
+++

**iptables**  Linux çekirdeği tarafından sağlanan  _Netfilter_ paketinin
çatısında bulunan bir komut satırı aracıdır. Bu araç Linux çekirdeğinin
güvenlik duvarında (firewall) bulunan kuralları değiştirmemize, düzeltmemize
ve yeni kurallar eklememize yardımcı oluyor. Yani iptables bir firewall görevi
görüyor. Basitçe sistemimizden internete çıkış yaparken veya sistemimize
dışarıdan giriş yapılırken kontrol edilecek bazı durumlar var ise bunları
tanımlamamıza, bunları bir tabloya kurallar dizisi şeklinde kaydetmemize
yarıyor.

Bu aracı kullanabilmek için yönetici yetkilerine sahip olmanız gerekiyor. Yani
ilgili sistemde root bilgilerine sahip olmanız ya da superuser yetkisine sahip
olan bir kullanıcınız olması gerekiyor.

iptables IPv4 güvenliği için kullanılan bir araç. Linux’da IPv6 ve IPv4
güvenlikleri ayrı şekilde sağlanıyor. IPv6 güvenliği için  **ip6tables**
kullanabilirsiniz.

## iptables’e nasıl ulaşırım?

**iptables** Linux dağıtımlarının çok büyük çoğunluğunda hazır olarak geliyor.
Eğer dağıtımınız başka bir firewall uygulaması kullanıyor ise, iptables
yüklemek için dağıtımınızın dökümasyonlarında paketi nasıl yükleyeceğinizi
araştırmalısınız.  **iptables** genel olarak Linux sistemlerinde
`/usr/sbin/iptables` veya `/sbin/iptables`  dizininde bulunuyor.
**iptables**'in komut yapısını (syntax) ve parametrelerini görmek için
`man iptables` yazabiliriz.

## Kullanım

**iptables**’in bir kurallar dizisinin bulunduğu bir tablo olduğunu
söylemiştik. Bu kuralları görüntülemek için `-L` veya  `--list`
parametrelerinden birini kullanacağız. Kuralları görüntüleyelim:

```shell
root@suayip-lnx:~root@suayip-lnx:~# iptables -L
Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
```

Eğer siz de benim gibi herhangi bir kural eklemediyseniz muhtemelen böyle bir
tablo karşınıza gelecek. Gördüğünüz gibi tabloda 3 zincir (chain) var.
_INPUT_,  _FORWARD_ ve  _OUTPUT_. Bu zincirler adlarından anlaşıldığı üzere
sırayla giriş, yönlendirme ve çıkış durumlarında neler yapılmasını gerektiğini
listeliyor. Her zincirin bir policy değeri var. Şuan hepsinin değeri ACCEPT.
Hemen bu policy’lerden birini değiştirip internete çıkış yapmamızı engelleyen
bir örnek yapalım. Öncelikle tabloda kayıtlı olan kuralların bir çıktısını
`-S` parametresi ile alalım:

```shell
root@suayip-lnx:~# iptables -S
-P INPUT ACCEPT
-P FORWARD ACCEPT
-P OUTPUT ACCEPT
```

Gördüğünüz üzere tüm policy’ler şuan ACCEPT durumunda. İnternete çıkış yapmayı
engellemek istediğim için OUTPUT policy değerini DROP olarak değiştirmemiz
gerekiyor. Policy ayarını değiştirmek için `-P` parametresini kullanacağız:

```shell
root@suayip-lnx:~# iptables -P OUTPUT DROP
```

Bu komutu çalıştırdıktan sonra internete çıkış yapamaz hale geleceğiz. Teyit
etmek için Google amcaya ping atmayı deneyelim:

```shell
root@suayip-lnx:~# ping google.com
ping: google.com: Name or service not known
```

Gördüğünüz gibi artık sistemimizin internete çıkışı engelleniyor. Kurallar
listesine bakarak OUTPUT’un policy değerinin DROP olduğunu da görebilirsiniz.
Tekrar internete ulaşmak için aynı şekilde OUTPUT’un policy değerini ACCEPT
olarak değiştirin. Zincirlerin policy’leri sadece ACCEPT ve DROP olabilir.

Basit olarak internete çıkışımızı veya internetten veri gelişini engelledik.
Fakat bu kullanımlar pek işimize yaramayacak. Sonuç olarak spesifik olarak
giriş ve çıkışları engellemek istiyoruz. Bu durumda tabloya eklemeler yapmaya
başlayabiliriz. Örneğin spesifik bir ip adresinden gelen istekleri engellemek
istiyoruz. Tabloya şu şekilde bir ekleme yapabiliriz:

```shell
iptables -A INPUT -s 216.58.206.174 -j DROP
```

Parametreleri tek tek incelersek:

`-A` sonrasında gelecek zincire ekleme yapmak istediğimizi söylüyoruz (append).

`-s` kaynak belirtiyoruz. bu kaynak herhangi bir ip adresi veya hostname
olabilir.

`-j` kuralın ne yapacağını belirtiyoruz. bu örnekte DROP etmesini söyledik
örneğin.

Bu parametre için aynı zamanda REJECT de kullanabilirdik. İkisi arasındaki
farkı açıklamak gerekirse, DROP karşı tarafa herhangi bir şey söylemiyor. Öte
yandan REJECT karşı tarafa bağlantının reddedildiğini dolayısıyla bir firewall
korumasının olduğunu belirtiyor. Bu yönden bakınca DROP kullanmak her zaman
daha makbul gelebilir fakat doğru değil. Kullanım alanlarınıza göre DROP veya
REJECT 'den hangisini kullanacağınızı internetten araştırmalısınız.

Komutu sorunsuz bir şekilde çalıştırdıktan sonra `iptables -L` komutu ile
listemize bakarsak kullandığımız zincirin altına yeni bir kural eklenmiş
olduğunu göreceksiniz. Kuralımızı test edelim. Kullandığım IP adresi Google'ye
ait ip adreslerinden biriydi. Yine üstteki şekilde ping attığım zaman, paket
gönderdiğim halde kendim paket alamıyorum:

```shell
--- 216.58.206.174 ping statistics ---
11 packets transmitted, 0 received, 100% packet loss, time 10216ms
```

Başarılı bir şekilde spesifik bir adresten gelen, girişleri veya o adrese
çıkışları engelleyebildik. Peki oluşturduğumuz kuralları nasıl silebiliriz? Ben
2 yöntem vereceğim. Birinci yöntemimiz spesifik bir kuraları silmeye yönelik.
Diğer yöntem ise tüm kuralları toptan silmek için. İlk yöntem için `-D`
parametresini kullancağız. Kural eklerken kullandığımız `-A` parametresi
yerine `-D` koymak yeterli olacak:

```shell
root@suayip-lnx:~# iptables -D INPUT -s 216.58.206.174  -j DROP
```

Listeye tekrar bakarsanız kuralın silinmiş olduğunu göreceksiniz. Peki ya
yazdığımız kuralı unuttuysak ne olacak? Üstte kayıtlı olan kuralların çıktısını
`-S` komutu ile alabileceğimizi zaten biliyoruz. Buradan aldığımız çıktıyı
direkt kopyalayarak parametreyi yine `-D` olarak değiştirmemiz yeterli
olacaktır.

Eklediğimiz kurallar çok ise ve tek tek silmek istemiyorsak işimiz yine basit.
`-F` parametresini kullanarak tüm kayıtları silebiliriz. Eğer spesifik bir
zincirdeki kayıtların tamamını silmek istiyorsak  `-F`parametresine bu zinciri
belirtebiliriz. Örnek kullanımlar:

```shell
root@suayip-lnx:~# iptables -F ## tüm zincirlerdeki kurallar
root@suayip-lnx:~# iptables -F INPUT ## sadece INPUT zinciri
```

## Birkaç örnek kullanım

Yukarıda iptables’in ne olduğunun ve nasıl bir yapısının olduğunu hakkında
biraz bilgi edindik. Aslında bu aşamadan sonra yapacağımız şey kullanım
amacımızı belirlemek ve bu doğrultuda gerekli olacak parametreleri manuel
sayfasından araştırmak olacak. Şimdi ilham vermesi açısından birkaç kullanım
görelim:

**a)**  SSH portundan gelen istekleri DROP eden bir kural, burada kaynak
yerel ağ olarak belirtilmiş. Yani yerel ağdan kimse bu makineye SSH ile
bağlanamaz. Spesifik bir IP adresi de tercih edilebilirdi:

```shell
iptables -A INPUT -p tcp --dport ssh -s 192.168.1.1 -j DROP
```

**b**) Geldiği yer farketmeksizin her türlü SSH bağlantısını reddeden kural,
`--dport`  için herhangi bir port da belirtebilirdik.

```shell
iptables -A INPUT -p tcp --dport ssh -j DROP
```

Yukarıdaki örnekten yola çıkarak yararlı başka bir kural elde edebiliriz.
Örneğin zinciri OUTPUT olarak değiştirip portu 80 yapabiliriz. Bu sayede
güvenlik sertifikası olmayan sitelere erişemezdik.

**c)**  Makinemize yapılacak olan girişleri dakikada 25'e limitleyen bir kural.
DoS ataklarını engellemek için kullanılabilir:

```shell
iptables -A INPUT -p tcp --dport 80 -m limit\
         --limit 25/minute --limit-burst 100 -j ACCEPT
```

Yukarıda gördüğünüz kuralda `-m` parametresi kullanılmış. Bu parametre bir
eklentiye işaret ediyor. Iptables'in çokça eklentisi de bulunmakta. Sonradan
gelen limit parametreleri de bu eklentiyle ilgili. Fazla sayıda eklenti ve
yüzlerce parametre olduğu için bunları tek tek açıklamayacağım. `man` sizin
dostunuz!

## Kuralları kalıcı hale getirmek

Kuralları ekleyip pek çok değişiklik yapıyoruz, fakat bu değişiklikler sistemin
o anki oturumu için geçerli. Bu değişiklikleri kalıcı hale getirmek istiyorsak
çalıştırmamız gereken bir komut var. Bu komut dağıtıma göre değişebilir. Ben
Ubuntu 18.04 kabuğunda şu şekilde çözdüm ve şöyle bir çıktı aldım:

```shell
root@suayip-lnx:~# /sbin/iptables-save
# Generated by iptables-save v1.6.1 on Tue Aug 14 17:29:44 2018
*filter
:INPUT ACCEPT [9:5501]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [6:464]
-A INPUT -s 216.58.206.174/32 -j DROP
COMMIT
# Completed on Tue Aug 14 17:29:44 2018
```

Red Hat / CentOS kullanıcıları için:

```shell
/sbin/service iptables save
```

veya

```shell
/etc/init.d/iptables save
```

## Sonuç

Belki gücünü görmüş olmayabilirsiniz ama iptables gerçekten güçlü bir araç.
Yazı genel olarak iptables hakkında size kaba bir bilgi vermiştir diye
düşünüyorum. Aslında erbabı için daha anlatılacak tonla şey var. Ben ancak
kendi bilgim ve araştırmalarım doğrultusunda elde edebildiğim anlaşılır ve
işe yarar bazı kilit bilgileri vermeye çalıştım. Bu kaba bilgilerden yola
çıkarak neler yapabileceğinizi keşfedeceğinizi öngörüyorum. Kolay gelsin.
