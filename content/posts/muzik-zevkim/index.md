+++
title = "şuayip'in müzik zevkini inceliyoruz"
date = 2026-10-28T07:00:00Z
tags = ["personal", "turkish"]
slug = "muzik-zevkim"
+++

{{< youtube hgpXSoWBAVA >}}

Kendi sitende ne diye kendine ucuncu sahisla hitap ediyorsun, diye soruyor
olabilirsiniz, hemen kendimi clarify edeyim -- ben şuayip'in personal AI
asistani Burcu.

Saka saka, bu yazi tamamen bir insan (şuayip) tarafindan yazilmistir. Hatta
sizi ikna etmek icin bu yaziyi AI ile deasciify etmekten bile vazgectim;
usendigimden falan degil.

Bu arada, ozenip daha guzel kod yazacagim, kisayol tuslarini remap etmekle
ugrasmayacam diye US layout klavye'ye alisirsaniz amerika'da buyumus ama
bilingual turkcesi olan insan aksani gibi bi parmak aksani falan oluyor,
anladiniz onu. sıkıntı gibi kelimeleri lugatinizdan cikarmaniz falan da
gerekiyor, artik 'sorun' insanisiz.

---

Neyse, bildiginiz uzere muzik dinlemeyen insanlar disinda butun insanlar muzik
dinlerler. Muzik dinleme ihtiyacimizi artik KRAL TV gibi yerlerden gideremiyoruz
cunku modernlestik ve evimizde televizyon yok, ayrica yagmurda cekmiyor (evet)
ve 480p. TRT zaman zaman bizi mutlu etse de halkin tamamina hitap eden bir
yayin sekli benimsedigi icin hard rock ve klasik muzik yayinlari 23.00-01.00
arasinda (ve inanir misiniz art arda) yayinliyorlar. O siralarda da
doomscrolling ve benzeri gerekli aktivitelerle mesgul oldugumuz icin kaciriyoruz.

Dolayisiyla hepimizin (muzik dinleyenler haric) dijital platformlarda uyelikleri
var, hem bu platformlarda sarki dinleyerek artistlerimize de para kazandirmis
oluyoruz. Ornegin ben tek basima Spotify'da [Özlem Temiz](https://open.spotify.com/artist/622LJ8QUml48wj43yAJrxv?si=voB-zS-YSmaRL1CYb3ENGw) in 10 aktif
dinleyicisinden biri olarak en azindan 20 TL falan kazandirmisimdir?

Bugun spesifik olarak Spotify ile ilgili yaziyorum cunku zamaninda onu
begenmisim. Biliyorsunuz ki Spotify her yil sonunda bizi wrapped olayiyla
senlendiriyor, sanki butun yil dinledigimizi bilmezmiscesine bize o sarkiyi
47 kere dinledigimizi soyluyor ve bu konuda cok mutlu oluyoruz.

Bu mutluluk konusunda kafaya takmis olanlar bilecektir ki Spotify belli
istatisiklerinizi haftalik aylik ve yillik (rolling) olarak tutuyor. Bunlari
expose ettigi API'ler da var. [Bu](https://www.statsforspotify.com) gibi
sitelerden bu istatikleri gorebiliyorsunuz.

Bazi insanlar ne zaman, nasil, nerede ve niye muzik dinlediklerini o kadar
detayli gormek istiyorlar ki [last.fm](https://www.last.fm) tarzi ucuncu parti
uygulamalara dijital platformlara baglayarak sarki akislarini kaydediyorlar.
Tabi last.fm geriye donuk muzik aliskanliklarinizi bilemiyor, ancak onu
kullanmaya basladiktan sonra naptiginizi gorebiliyorsunuz.

Avrupa Birligi'nin faydalarina gelecek olursak, Spotify'da dinleme
gecmisininizin tamamini indirebiliyorsunuz. Ve oyle dandik bir bir veri de
degil. Gayet gayet, hangi sarkiyi ne zaman kac saniye dinlemissiniz hepsini
gorebiliyorsunuz. Datanin icinde, IP adresiniz, user agentiniz bile var, bunlar
hep onemli bilgiler 😉 Gelen data o kadar detayli ki, bir sarkiyi skip edip
etmediginiz bile anlasiliyor. Soyle bır CSV header'ı var:

```csv
ts,username,platform,ms_played,conn_country,ip_addr_decrypted,
user_agent_decrypted,master_metadata_track_name,
master_metadata_album_artist_name,master_metadata_album_album_name,
spotify_track_uri,episode_name,episode_show_name,spotify_episode_uri,
reason_start,reason_end,shuffle,skipped,offline,offline_timestamp,
incognito_mode,ip_addr,audiobook_chapter_title,audiobook_chapter_uri,
audiobook_uri,audiobook_title
```

Bu datayi indirmek icin surayi ziyaret edebilirsiniz, 'ayrıntılı çevrimiçi
dinleme geçmişi' secip veri talebinde bulunmaniz gerekiyor.

https://www.spotify.com/tr-tr/account/privacy/

Genelde verileriniz bir hafta icinde geliyor. Eger sansliysaniz ertesi
gun bile gelebilir.

Bu verilerinizi aldiktan sonra hemen DuckDB'ye bi aktariverin. Dosyalar parca
parce halinde geliyor. Ornegin `Streaming_History_Audio_2017-2018_0.json` o yil
abartmissaniz yillar da kendi icinde bolunebiliyor. Tavsiyem butun JSON'lari
bir CSV icinde aggregate edip DuckDB'da data source olarak import etmeniz.

Yani buradan sonra ne denebilir bilmiyorum, olasiklar sonsuz. Sirf bu veriyi
alip otomatize bir sekilde chart'lar, ucan kacan seyler yapmayi dusundum ama
usendim -- siz bir arastirin belki yapilmistir? guzel bir haftasonu projesi.
Tabii burada fikirler onemli. Simdi benim cikardigim bikac query'e bakalim.

---

**en cok dinlenen albumler**

```sql
select master_metadata_album_album_name               as name,
       count(*)                                       as count,
       round(sum(cast(ms_played as int) / 3.6e+6), 2) as total_playtime
from hist
group by master_metadata_album_album_name
order by total_playtime desc
limit 50;
```

```csv
name,count,total_playtime
Sehir Rehberi,5441,167.88
Olsun,5214,103.41
Dünya Yalan Söylüyor,3165,102.79
```

**2025 yilindan once once dinleyip begenmedigim(?) ama sonradan baya bayildigim
sarkilar:**

```sql
with plays as (select spotify_track_uri,
                      master_metadata_album_artist_name,
                      master_metadata_track_name,
                      sum(case
                              when substr(ts, 1, 4) < '2025'
                                  then cast(ms_played as int)
                              else 0 end) as ms_before_2025,
                      sum(case
                              when substr(ts, 1, 4) = '2025'
                                  then cast(ms_played as int)
                              else 0 end) as ms_2025
               from hist
               group by spotify_track_uri)

select master_metadata_album_artist_name || ' - ' || master_metadata_track_name as name,
       round(ms_before_2025 / 60000.0, 1)                                       as minutes_before_2025,
       round(ms_2025 / 3.6e+6, 2)                                               as hours_2025
from plays
where ms_before_2025 > 0
  and ms_before_2025 < 30 * 60000
  and ms_2025 > 1 * 3.6e+6
order by hours_2025 desc
limit 100;
```

```csv
name,minutes_before_2025,hours_2025
Tame Impala - The Less I Know The Better,0,5.91
The Police - Message In A Bottle,0.4,4.67
Fleetwood Mac - Dreams - 2004 Remaster,0.1,4.56
```

**surekli surekli skip ettigim sarkilar, bunun icin cikan sonuclarimdan parca
vermicem 😇**

```sql
select master_metadata_album_artist_name || ' - ' || master_metadata_track_name as name,
       count(*)                                                                 as plays,
       round(avg(cast(ms_played as int)) / 1000, 1)                             as avg_seconds,
       sum(case when cast(ms_played as int) < 30 * 1000 then 1 else 0 end)      as skips_under_30s,
       round(
               sum(case when cast(ms_played as int) < 30 * 1000 then 1 else 0 end)
                   * 1.0 / count(*),
               2
       )                                                                        as skip_ratio
from hist
group by spotify_track_uri
having plays >= 10
   and skip_ratio >= 0.7
order by skip_ratio desc, plays desc
limit 100;
```

**geldigi zaman asla salmadigim silip supurdugum sarkilar (yukarinin tersi)**

```sql
select master_metadata_album_artist_name || ' - ' || master_metadata_track_name as name,
       count(*)                                                                 as plays,
       round(avg(cast(ms_played as int)) / 1000, 1)                             as avg_seconds,
       sum(case when cast(ms_played as int) < 30 * 1000 then 1 else 0 end)      as skips_under_30s,
       round(
               sum(case when cast(ms_played as int) < 30 * 1000 then 1 else 0 end)
                   * 1.0 / count(*),
               3
       )                                                                        as skip_ratio
from hist
group by spotify_track_uri
having plays >= 10
   and skip_ratio <= 0.25
order by skip_ratio asc, plays desc
limit 100;
```

```csv
name,plays,avg_seconds,skips_under_30s,skip_ratio
'Til Tuesday - Winning the War,20,229.3,1,0.05
Queen - Hammer To Fall - Remastered 2011,12,223.5,1,0.083
Mirkelam - Tavla,45,240.3,5,0.111
```

dusundugum ama bana pek eligible olmadigi veya usendigim icin yazmadigim seyler:

- yurt disina cikinca degisen muzik icin bir delta analizi
- telefonda/bilgisayarda dinlenen muzigin dagilimi
- eger duzenli bir saatte gym'e gittiginizi biliyorsaniz ya da benzer duzenli
  aktiviteniz varsa o aktivite sirasindaki muzik habitleriniz
- gece/gunduz - yaz/kis gibi kategorilere sarki/artist dagilimi

---

şuayip istatiksel olarak icimizi cok serinlettin. peki nerede bu duygusal
analiz?

- Bana 7 ekim 2021 gece 3'te neden Destan - Cilveloy spam'ladigimi kim aciklayabilir?
- Duran Duran ile Nil Karaibrahimgil'i nasil ayni playlistte erittigimi bana anlatir misin?

gibi sorulariniz oldugunu biliyorum. Bu sorulari cevaplamak icin our trusted
agent'imize gitmemiz gerek (Burcu gel buraya). Oncelikle butun playlistinizin
bir snapshotunu almaniz lazim. [Ne sanslisiniz ki bunun icin bir tool var](https://github.com/realsuayip/spotify-backup).
Bu toolun cikarttigi CSV outputu komple agent'a yukleyin, her bir field context
icin onemli.

Sadece bu playlist export datasiyla bile detayli birkac analiz yapabilirsiniz.
Ozellile detayli genre aliskanlilarinizi ve artist discovery processinizi
gorebilirsiniz. Ornegin sarkilarin eklenme tarihine gore hangi artisti surekli
(consistent) seviyor anliyorsunuz. zamanla muzik zevkiniz degistiyse bunu rahat
bir sekilde fark edebiliyorsunuz.

yapabileceginiz bir sey buna ek olarak yukaridaki veri setinde belirli sorgular
calistirip CSV'nizi annotate etmek. ornegin 2026_listen_hours, 2025_listen_hours
gibi field'lar ekleyerek yillik analizler alabilirsiniz.

buna ek olarak cesitli istatislileri aggregate edip yine export CSV yaninda
verebilirsiniz. kendinizi cok tembel hissediyorsaniz, agent'a gerekli olan
veri setlerini de kendisinin SQL yazarak cikarmasini isteyebilirsiniz. Bundan
sonra yapmaniz tek gereken sey, yukaridaki gibi sacma sapan sorular sorarak
bilincaltinizin derinliklerine inmek.

---

ey şuayip, ben senin muzik zevkini merak etmistim ama onla ilgili pek bisey
koymamissin diyor olabilirsiniz (wink wink orneklerdeki veri setleri dogru olmayabilir).
su an Burcu cok haril haril bir sekilde calisiyor, bu detayli
deep-down, find-yourself temali ve deep emotional sensation yaratacak kisisel
analizimi yakin zamanda burada (hemen asagida) paylasacagim.
