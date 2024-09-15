+++
title = "Modern Django Proje Yapısı & Docker Kullanımı"
date = 2024-09-15T21:36:04Z
tags = ["django", "docker", "turkish"]
slug = "modern-django-docker-kullanimi"
+++

Bu yazıda, modern (bleeding edge) bir Django projesinin nasıl gözüktüğünü,
hangi tool'ların kullanıldığını ve nasıl Docker ile containerize edileceğini
göstereceğim.

Yazının sonunda Docker ile development ve production ortamları için birer
configuration oluşturmuş olacağız, sadece tek komutla bu ortamlarda projemizi
ayağa kaldırabileceğiz.

Yine projemizi lint etmek, formatlamak, ve type checking yapmak için mevcut
olan en yeni ve en janjanlı yöntemleri bu yazımda anlatıyorum.

## Dizin Yapısı

Öncelikle projemizin dizin yapısından başlayalım, projeniz temelde şu şekilde
gözükmeli:

```text
.
├── conf
├── docker
├── docs
├── manage.py
├── pyproject.toml
├── tests
├── twitter
│   ├── __init__.py
│   ├── apps.py
│   ├── gateways
│   │   ├── asgi.py
│   │   └── wsgi.py
│   ├── locale
│   ├── settings
│   │   ├── __init__.py
│   │   ├── apps.py
│   │   ├── django.py
│   │   └── test.py
│   ├── static
│   ├── templates
│   ├── tweet
│   │   ├── __init__.py
│   │   └── apps.py
│   ├── urls.py
│   └── utils
│       ├── cache.py
│       ├── file.py
│       └── mailing.py
└── uv.lock
```

Şimdi bu dizinlerin ve dosyaların hepsini tek tek inceleyelim. Bu örnekte
projemizin ismi `twitter`.

### conf

Bu dizinde projemize ait environment variable'ler dolayısıyla bütün ayarları
içeren dosyalar bulunacak, `DEBUG`dan tutun `STRIPE_API_KEY`e kadar her
şey yani.

Projemizi Django tarafında configure ederken hiçbir environment variable'a
default veya boş değer atamıyoruz. Bütün ayarlar buradan çekilmeli, çekilemezse
proje hata vermeli.

### docker

Projemize ait bütün `Dockerfile`ler bunun yanında compose dosyaları ve diğer
servislerin (örneğin `nginx`) ayarlarını içerecek dosyalar bu dizin altında
olacak.

### docs

Projemiz için yazmayı unutmadığımız documentation burada olacak. Dokümantasyonu
`reStructuredText` kullanarak yazıyoruz ve `sphinx` kullanarak compile
ediyoruz.

### manage.py

Django uygulamamızı başlatacak ve management komutlarını yönetecek entrypoint.

### pyproject.toml

Projenin kütüphane gereksinimlerini, linter, formatter ve type checking dahil
envai çeşit tool'un configuration'unu içerecek dosyamız.

### uv.lock

Projenin kütüphanelerini kilitlediğimiz dosya. Projede `uv` kullanıyoruz.

### tests

Bütün testler bu dizin altında olacak. Testlerimizi proje içindeki dizinlerde
yazmıyoruz, bu sayede test discovery daha hızlı çalışıyor hem de bütün testler
daha derli toplu duruyor; 2 saat projeyi deşmek durumunda kalmıyoruz.

### twitter

Bu projemizin ana dizini, bu dizin içinde asıl Python dosyalarımız, Django
modellerimiz vesaire olacak. Şimdi bu projenin altındaki dosyalara ve dizinlere
bakalım.

#### apps.py

Sürpriz! Projemizin kendisi de bir Django application. Normalde bir Django
projesini başlattığınız zaman herhangi bir app oluşmaz; Not on my watch!

Bu app'te proje geneli herhangi bir app'e uyduramadığımız genel logic'leri
koyacağız. Örneğin benim kullandığım `ProjectVariable` isimli bir modelim var,
bu model her app'in kullanabileceği dinamik ayarları içinde tutuyor.

Bu tarz işleri yapmak için insanlar genelde `core` tarzında bir app daha
oluştururlar ama ben bu yöntemi daha sağlıklı buluyorum.

#### urls.py

URL'ların giriş yeri, yani `ROOT_URLCONF`.

#### settings

Burada projemize ait Django settings file olacak. Alt dizindeki dosyalara
bakacak olursanız, dev ve prod ayrımı yok zira o ayrım environment'tan gelecek.

Django ayarlarını içeren bir dosya `(django.py)`, üçüncü parti ve projenin
kendisine ait olacak ayarları içeren bir diğer dosya `(apps.py)` var. Bunlar
daha sonra `__init__.py` üzerinden birleştiriliyor ve nihai
olarak `twitter.settings` bizim Django settings file'miz oluyor.

Ek olarak test ortamı için environment yerine ek bir dosya var.

#### locale, static, templates

Dil dosyaları, statik dosyalar ve template'lerin olacağı yer. Ek olarak diğer
app'lerde bu dosyalardan hiçbiri olmamalı, hepsi burdaki iç dizinlerde
bulunmalı.

#### utils

Projenin sağında solunda kullanabileceğimiz yardımcı fonksiyonları içeren
modül.

#### gateways

Kullanacağınız gateway interface'lerini içeren modül. Eğer tek bir tane
kullanıyorsanız, bu dizini silebilirsiniz.

#### tweet

Bu arkadaş da örnek bir app.

## Tooling

Şimdi projemiz için kullanacağımız tool'lara bakalım.

### uv

Projemizdeki kütüphaneleri yönetmek için bu kütüphaneyi kullanacağız. Kütüphane
gereksinimlerimizi `pyproject.toml` dosyasında belirteceğiz ve daha sonra bu
kütüphaneleri `uv.lock` yoluyla kilitleyeceğiz. Bu sayede sürekli aynı
kütüphaneleri kullandığımzdan emin olacağız.

Kütüphanelerimizden bazılarını sadece development ortamı için, bazılarını da
sadece production ortamı için kuracağız. Bu ayrımı yine configuration
dosyasında yapacağız. Şimdi örnek bir configuration görelim:

```toml
[project]
name = "twitter"
version = "0.1.0"
# 1
requires-python = "==3.12.*"
# 2
dependencies = [
    "django ~= 4.2",
    "djangorestframework ~= 3.15",
    "asgiref ~= 3.8",
    "channels ~= 4.1",
    "channels-redis ~= 4.2",
    "drf-spectacular",
    "drf-nested-routers ~= 0.94",
    "django-filter ~= 24.3",
    "django-cors-headers ~= 4.4",
    "django-oauth-toolkit ~= 3.0",
    "django-two-factor-auth ~= 1.17",
    "django-storages ~= 1.14",
    "django-ipware ~= 7.0",
    "django-celery-beat ~= 2.7",
    "django-stubs-ext ~= 4.2",
    "django-widget-tweaks ~= 1.5",
    "sorl-thumbnail ~= 12.10",
    "celery ~= 5.4",
    "psycopg[c] ~= 3.2",
    "redis ~= 5.0",
    "hiredis ~= 3.0",
    "Pillow ~= 10.4",
    "python-magic ~= 0.4",
    "boto3 ~= 1.35",
    "phonenumbers ~= 8.13",
    "sentry-sdk ~= 2.14",
    "envanter ~= 1.2",
]

# 3
[project.optional-dependencies]
dev = [
    # Testing
    "factory-boy",
    "coverage",
    # Typing
    "mypy",
    "django-stubs",
    "djangorestframework-stubs",
    "types-oauthlib",
    "types-Pillow",
    "celery-types",
    # Misc
    "daphne",
    "django-debug-toolbar",
    "ipython",
    "tblib",
    "watchfiles",
]
prod = [
    "gunicorn ~= 23.0",
    "uvicorn ~= 0.30",
    "uvloop ~= 0.20",
    "httptools ~= 0.6",
    "websockets ~= 13.0",
]
```

1. Projemizde kullanacağımız Python versiyonlarını belirtiyoruz. Bu hangi
   kütüphanelerin nasıl yükleneceği konusunda önemli olduğu için mutlaka
   belirtmeliyiz.
2. Projemizin kütüphanelerini belirtiyoruz. Burda genel olarak major ve minor
   versiyonlarını sabit tutup, patch versiyonlarını salık vermek iyidir. Bu
   sayede patch'leri kolay bir şekilde uygulayabiliriz.
3. Dev ve prod ortamına özel kütüphaneleri belirtiyoruz. Dev ortamında
   kütüphane belirtirken versiyon belirtmiyoruz, her zaman en güncel olanları
   seçiyoruz. Bu sayede her zaman en yeni development araçlarını kullanıyor
   olacağız. Eğer bir araç yeni versiyon'unda sorun çıkarıyorsa ve o sorunu
   çözemiyorsak, bu sorun çözülene dek versiyonu sabitleyebiliriz.

### ruff

Projemizi lint'lemek ve formatlamak için bu tool'u kullanıyoruz. Ruff
ayarlarımızı `pyproject.toml` içinde nizamı bir şekilde ayarlıyoruz. Örneğin:

```toml
[tool.ruff]
fix = true
show-fixes = true
target-version = "py312"
line-length = 88

[tool.ruff.lint]
fixable = ["I"]
select = [
    "E", # pycodestyle errors
    "W", # pycodestyle warnings
    "F", # pyflakes
    "C", # flake8-comprehensions
    "B", # flake8-bugbear
    "RUF", # Ruff-specific
    "C4", # flake8-comprehensions
    "C90", # mccabe
    "I", # isort
    "N", # pep8-naming,
    "BLE", # flake8-blind-except
    "DTZ", # flake8-datetimez
    "DJ", # flake8-django
    "FA", # flake8-future-annotations
    "G", # flake8-logging-format
    "T20", # flake8-print
    "RET", # flake8-return
    "SIM", # flake8-simplify
    "PTH", # flake8-use-pathlib
    "ERA", # eradicate
    "PL", # pylint
    "PERF", # perflint
]
ignore = ["B904", "RUF012", "SIM105", "PTH123", "PLR2004", "PLR0913", "PLR0911"]

[tool.ruff.lint.isort]
combine-as-imports = true
section-order = [
    "future",
    "standard-library",
    "django",
    "rest_framework",
    "third-party",
    "first-party",
    "local-folder",
]

[tool.ruff.lint.isort.sections]
django = ["django"]
rest_framework = ["rest_framework"]
```

Burda dikkat edilmesi noktalar şunlar:

* Django'dur canı çeker, query yazmak daha kolay oluyor falan deyip satır
  uzunluğunu 120 yapmayın. Kod yukarıdan aşağı doğru okunur; hele Python'da 120
  karakterle dünyayı dolaşabiliyorsunuz. O yüzden mütevazi olun ve 88'de kalın.

* Django ve REST framework import'ları sıkça kullanıldığı için bunları spesifik
  olarak ayrı gruplandıran ve öne çıkaran bir import formatting ayarı var.

### mypy

Projemizi type check etmek için `mypy` kullanacağız. Django bağlamında zaten
başka alternatifimiz yok. `django-stubs`, `djangorestframework-stubs` ve
`celery-types` olmazsa olmaz eklentilerimiz.

Lütfen Django projelerimize type hint ekleyelim, erinmeyelim. Örnek bir
configuration:

```toml
[tool.mypy]
plugins = [
    "mypy_django_plugin.main",
    "mypy_drf_plugin.main",
]
strict = true
ignore_missing_imports = true
allow_subclassing_any = true

[tool.django-stubs]
django_settings_module = "twitter.settings"
```

`strict = true` var görüyorsunuz değil mi? Öyle havadan mypy kullanmayalım.

### coverage

Test yazarken nereleri cover etmişiz, nereleri atlamışız görmemize yarayan bir
araç. Ben şahsen bu arkadaş dışında herhangi bir test komutu kullanmıyorum.

Bundan kastım `pytest` kardeşimiz ve onun eklentileri. Django'nun kendi
sağladığı test runner ve onun özellikleri gayet yeterli. `pytest-django` bu
özelliklerin tamamını karşılamıyor. Öte yandan `pytest` nispeten yavaş
çalışıyor.

### watchfiles

Kod değişiklikleri olduğu zaman Django server'inin tekrar başlaması için bu
tool'u kullanıyoruz. Django'nun default olarak içinde bulunan mekanizmadan daha
hızlı çalışıyor ve daha az kaynak tüketiyor.

### django-debug-toolbar

Benim genelde SQL optimizasyonu yapmak için kullandığım, bir ton başka
feature'ları da olan güzel bir debug aracı.

### sphinx

Dokümantasyon ile ilgili her şeyi bu araç üzerinden gerçekleştiriyoruz. Zaten
bir Python standardı kendisi.

Ek olarak belirtmeliyim, dokümantasyona ait tüm kütüphaneler için ayrı bir lock
dosyası `docs` dizini altında olmalı, ana projemizde değil. Bu sayede
projemizin sadece dokümantasyonunu build ederken diğer kütüphaneleri içeri
almıyoruz.

## Docker Kullanımı

İlk olarak `docker` dizini altında kullanacağınız yapıyı inceleyelim:

```text
.
├── dev
│   ├── common.yml
│   ├── dev.Dockerfile
│   ├── dev.Dockerfile.dockerignore
│   └── docker-compose.yml
└── prod
    ├── django
    │   ├── common.yml
    │   ├── prod.Dockerfile
    │   └── prod.Dockerfile.dockerignore
    ├── docker-compose.yml
    └── nginx
```

`dev` ve `prod` olarak ayırdığımız dizinlerde her bir environment'e ait
`Dockerfile` ve compose dosyaları mevcut. Production versiyonunda her servis
için ayrı bir dizin var. Burada `nginx` için detaya girmeyeceğim ama yazının
sonunda vereceğim link'te dilerseniz onu da inceleyebilirsiniz.

Şimdi dev'den başlayalım.

### dev

`dev.Dockerfile` içeriğini inceleyelim:

```dockerfile
# 1
FROM ghcr.io/astral-sh/uv:python3.12-alpine

# 2
WORKDIR /app

# 3
ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    UV_COMPILE_BYTECODE=1 \
    UV_LINK_MODE=copy \
    PATH="/app/.venv/bin:$PATH"

# 4
RUN --mount=type=cache,target=/var/cache/apk \
    --mount=type=cache,target=/etc/apk/cache \
    apk update && apk add gcc musl-dev libpq-dev libmagic gettext

# 5
RUN --mount=type=cache,target=/root/.cache/uv \
    --mount=type=bind,source=uv.lock,target=uv.lock \
    --mount=type=bind,source=pyproject.toml,target=pyproject.toml \
    uv sync --frozen --extra dev

# 6
COPY .. .

# 7
ENTRYPOINT []
```

1. UV'nin `alpine` dağımını kullanacağız. Her zaman alpine kullanmaya
   çalışalım, zira neden olmasın? Hızlı build almak ve container size'i
   düşük tutmak için alpine birebir.
2. Container'da ana dizini `/app` yapalım. bütün uygulamayı burada tutacağız
   aynı zamanda `Dockerfile` bağlamında o dizine geçmiş olduk.
3. Burda Python'un genel geçer container environment'ları
   var:
    * `PYTHONUNBUFFERED` Django loglarını anlık yakalamak için
    * `PYTHONDONTWRITEBYTECODE` proje içinde `.pyc` dosyalarının oluşmaması
      için.
    * `UV_COMPILE_BYTECODE` uv ile yüklenecek kütüphanelerin `.pyc`
      dosyalarının oluşması için
    * `UV_LINK_MODE` `uv`nin cache'i düzgün kullanabilmesi için.
    * `PATH` virtual environment içinde bulunan executable'lerin (
      örneğin `django-admin`, `celery`) `PATH`e eklenmesi. Bu sayede bu
      komutları kullanmak için virtual environment activate etmemize gerek yok.
4. Sistem kütüphaneleri ve build alacak Python kütüphaneleri için (mesela
   psycopg) gereksinimleri yüklüyoruz. Yüklerken de bunların cache dizinleri
   cache mount olarak belirliyoruz. Bu sayede tekrar build alırken bu adımda
   değişiklik yapmadıysanız ama bir şekilde Docker layer invalidate olduysa (
   mesela yukarıda environment variable'lardan birini değiştirdiyseniz) hızlı
   bir şekilde cache'i kullanarak yükleme yapmak için.
5. uv ile lock dosyasından kütüphaneleri yükleyip virtual environment
   oluşturuyoruz. Bunu yaparken `pyproject.toml` ve `uv.lock` dosyalarını bind
   mount ediyoruz ki projemizde sadece bu dosyalara dokunursak Docker layer'i
   invalidate olsun. Ek olarak burada cache muhabbeti aynı yukardaki gibi,
   fakat burda çok daha iyi işe yarıyor çünkü kütüphane ekleme çıkarma
   yaptığınız zaman sadece istediğiniz kütüphane indiriliyor.
6. Proje dosyalarını mevcut dizine, yani `/app`e kopyalıyoruz.
7. Entrypoint'i sıfırla, burada kullandığımız image'in entrypoint'i `uv`, biz
   onu istemiyoruz.

Şimdi gelin `common.yml` dosyasını inceleyelim, bu dosya çalışacak Python
container'leri için ortak ayarlar içeriyor:

```yaml
services:
  python:
    image: twitter-python
    env_file: # 1
      - ../../conf/dev/django.env
      - ../../conf/dev/postgres.env
    restart: unless-stopped
    depends_on:
      - db
      - redis
    volumes:
      - ../..:/app # 2
      - /app/.venv # 3
```

1. Environment olarak ana dizindeki conf altındaki environment dosyalarını
   veriyoruz. Burada `postgres` servisimizle ortak environment olduğu için (
   database bağlantı bilgileri) onu da dahil ettik. Ek olarak, compose
   dosyalarında dosya yolları her zaman compose dosyasının bulunduğu yere göre
   olur, build context'e göre değil; buna dikkat edin.
2. Docker'daki `/app` dizini ile proje ana dizinine bir volume açıyoruz, bu
   sayede local'da yaptığımız değişiklikler anında container'a yansıyacak ve
   reload çalışacak.
3. İkinci adımda yaptığımız işlem yüzünden container içindeki virtual
   environment'i da volume'a almış olduk. Bu istediğimiz bir şey değil çünkü
   host makinesi ile o environment uyumlu olmayabilir. O yüzden bu dizinin
   dahil olmaması için böyle bir tanım yapıyoruz.

Şimdi asıl `docker-compose.yml` inceleyelim:

```yaml
volumes:
  pg-data:

services:
  db:
    container_name: twitter-postgres
    image: postgres:16.4-alpine
    user: postgres
    env_file: ../../conf/dev/postgres.env
    ports:
      - "5432:5432" # 1
    volumes: # 2
      - pg-data:/var/lib/postgresql/data

  redis:
    container_name: twitter-redis
    image: redis:7.4-alpine
    user: redis

  rabbitmq:
    container_name: twitter-rabbitmq
    image: rabbitmq:3.13-alpine

  web:
    container_name: twitter-web
    build: # 3
      context: ../..
      dockerfile: docker/dev/dev.Dockerfile
    command: python manage.py runserver 0.0.0.0:8000 # 4
    ports:
      - "8000:8000" # 5
    extends: # 6
      file: common.yml
      service: python

  celery-worker: &celery # 7
    container_name: twitter-celery-worker
    command: watchfiles --filter python "celery -A twitter worker -l info"
    depends_on:
      - db
      - redis
      - rabbitmq
    extends:
      file: common.yml
      service: python

  celery-beat:
    <<: *celery
    container_name: twitter-celery-beat
    command: celery -A twitter beat -l info
```

1. Postgres'in portunu host makinemize de açıyoruz, bu sayede PyCharm gibi IDE
   özellikleriyle tabloları görüntüleyip SQL query'ler çalıştırabiliriz.
2. Postgres'e özel named bir volume açıyoruz, bu sayede devamlı olarak local'de
   tuttuğumuz bir data'mız olacak.
3. Sadece bu Django container'a özel build command tanımlıyoruz. Burada Docker
   build context'i ve Dockerfile'i belirtiyoruz. Buradan üretilen image, diğer
   container'lar tarafından kullanılacak, örneğin Celery worker'imiz.
4. Django server'inin 8000 portundan çalışması için komutu belirtiyoruz.
5. İçerdeki 8000 portunu host makinemize yine 8000 portundan açıyoruz ki
   local'de browse edebilelim.
6. Önceden gördüğümüz ortak configuration'u bu şekilde belirtiyoruz.
7. Celery configuration'u isimlendiriyoruz bu sayede diğer Celery servisleri de
   sadece komut değiştirerek bunu kullanabilsin. Celery servisleri yine ortak
   configuration'u kullanıyor bu yüzden image kullanacak, build almayacak.

Burada ek olarak, `redis` ve `rabbitmq` servisleri var. Bu arkadaşlar default
olarak bazı portları içerdeki diğer servislere açıyorlar. Bu servislerin
ayarlarını da Django üzerinden yaptığımızda onlara erişebiliyoruz, tıpkı
Postgres'e eriştiğimiz gibi.

Docker'i dev ortamında kullanmak bu kadar kolay, standard docker-compose
komutlarını kullanabiliriz:

```shell
docker compose -p twitter -f docker/dev/docker-compose.yml up -d
```

Şimdi production ortamına bakalım.

### prod

Dockerfile dosyamızı inceleyelim:

```dockerfile
# 1
FROM ghcr.io/astral-sh/uv:python3.12-alpine AS builder

ENV UV_COMPILE_BYTECODE=1 \
    UV_LINK_MODE=copy

WORKDIR /app

RUN --mount=type=cache,target=/var/cache/apk \
    --mount=type=cache,target=/etc/apk/cache \
    apk update && apk add gcc musl-dev libpq-dev

RUN --mount=type=cache,target=/root/.cache/uv \
    --mount=type=bind,source=uv.lock,target=uv.lock \
    --mount=type=bind,source=pyproject.toml,target=pyproject.toml \
    uv sync --frozen --extra prod

COPY . .

# 2
FROM python:3.12-alpine

ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PATH="/app/.venv/bin:$PATH"

WORKDIR /app

# 3
RUN apk update && apk add --no-cache libpq libmagic tini

# 4
RUN addgroup -S django && adduser -S django -G django --disabled-password

# 5
COPY --from=builder --chown=django:django /app /app

# 6
USER django

# 7
ENTRYPOINT ["/sbin/tini", "--"]
```

1. Dev'de olduğu gibi kütüphanelerimizi yüklemek için `uv` image'ini kullanarak
   başlıyoruz. Burdaki tek fark, multi-stage build yaptığımız için bu aşamayı
   "builder" olarak isimlendirmek. Multi-stage build yapmamızın amacı hem
   gereksiz dosyaları egale ederek container size'ı düşürmek ve olabilecek en
   sade image'i elde etmek.
2. İkinci stage'a geçiyoruz, burada normal Python dağıtımına geçiyoruz,
   zira `uv` ile işimiz kalmadı.
3. Projemizin runtime'ında gerekli olan sistem paketlerini yüklüyoruz. Bu
   aşamada build dosyalarını image'da bırakmadığımıza dikkat edin.
4. Dev'de container'imiz root olarak çalışıyordu, production'da bunun yakışığı
   olmaz bu yüzden django isimli bir kullanıcı ve grup oluşturuyoruz.
5. İlk aşamada oluşturduğumuz virtual environment'i ve proje dosyalarını
   çekiyoruz, bunu yaparken de dosyaları django kullanıcısına atıyoruz.
6. django kullanıcısına geçiyoruz.
7. Entrypoint olarak `tini` aracını kullanıyoruz, bu sayede tek process'imiz
   düzgün bir şekilde çalışıyor. `tini` tam ne işe yarıyor merak ediyorsanız
   <https://github.com/krallin/tini> adresini ziyaret edin.

`common.yml` dosyamız ise fazla değişmedi, sadece burada volume'ları kaldırdık
zira artık ihtiyacımız yok:

```yaml
services:
  python:
    image: twitter-python
    env_file:
      - ../../../conf/prod/django.env
      - ../../../conf/prod/postgres.env
    restart: unless-stopped
    depends_on:
      - db
      - redis
```

`docker-compose.yml` dosyasını inceleyelim:

```yaml
volumes:
  pg-data:

services:
  db:
    container_name: twitter-postgres
    image: postgres:16.4-alpine
    user: postgres
    env_file: ../../conf/prod/postgres.env
    restart: unless-stopped
    volumes:
      - pg-data:/var/lib/postgresql/data

  redis:
    container_name: twitter-redis
    image: redis:7.4-alpine
    user: redis
    restart: unless-stopped

  rabbitmq:
    container_name: twitter-rabbitmq
    image: rabbitmq:3.13-alpine
    restart: unless-stopped

  nginx: # 3
    container_name: twitter-nginx
    build:
      context: ../..
      dockerfile: docker/prod/nginx/Dockerfile
    ports:
      - "443:443"
    restart: unless-stopped

  web:
    container_name: twitter-web
    build:
      context: ../..
      dockerfile: docker/prod/django/prod.Dockerfile
    command: > # 1
      gunicorn
      twitter.gateways.wsgi
      --bind 0.0.0.0:8000
      --workers 4
      --access-logfile '-'
    extends:
      file: django/common.yml
      service: python

  websocket: # 2
    container_name: twitter-websocket
    command: >
      gunicorn twitter.gateways.websocket
      --bind 0.0.0.0:7000
      --workers 4
      --worker-class twitter.utils.workers.UvicornWorker
      --access-logfile '-'
    extends:
      file: django/common.yml
      service: python

  celery-worker: &celery
    container_name: twitter-celery-worker
    command: celery -A twitter worker -l info
    depends_on:
      - rabbitmq
      - db
      - redis
    extends:
      file: django/common.yml
      service: python

  celery-beat:
    <<: *celery
    container_name: twitter-celery-beat
    command: celery -A twitter beat -l info
```

1. Production'da artık runserver kullanamayız, bu yüzden gunicorn command'ini
   veriyoruz.
2. Burada yine farklı bir Django container'i nasıl ayağa kalkar onu görüyoruz,
   mesela bu servis sadece WebSocket isteklerini karşılamak için oluşturulmuş.
3. Burada `nginx` servisini görüyoruz bu servisin detaylarına girmeyeceğim.
   Dikkat edersiniz dışarıya port açan tek servisimiz bu. İçerde nginx, Django
   container'lerini `7000` ve `8000` portlarından yakalayıp reverse proxy
   olarak görev görüyor.

## Özet

Bu yazıda Django'yu `uv` kullanarak nasıl paketleyip Docker yoluyla
dağıtabileceğimizi gösterdim. Ek olarak, bir Django proje yapısı ele aldım ve
Django ile kullanabileceğiniz güncel araçları listeledim. Umarım bu araçları
kullanırsınız ve janjanlı Django projenizi Docker'la yarınlara uğurlarsınız 🙂

Burada anlatılan yapıların detaylı uygulamasını içeren projeme şu adresten
ulaşabilirsiniz, burada detaylı `nginx` dosyaları da bulunuyor:

<https://github.com/realsuayip/asu>
