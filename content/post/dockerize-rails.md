---
title: Ruby on Rails ve Docker
date: 2018-03-04T11:38:34+30:00
authors:
  - Murat Bastas
---

Ben bu container, image vs işlerine çok geç başlamış olabilirim belki fakat yine de benimle aynı durumda, ya da benden daha bihaber durumda olan dolu arkadaşımız olabileceğini ve belki birisi denk gelir de bu yazıya docker kullanmayı öğrenirken aldığım notlara ihtiyaç duyabilir diye düşünerek yaklaşık 1 aylık docker uğraşlarımdan öğrendiklerimi ve bulduğum kaynakları paylaşmak istedim.

Bu blog üç bölümden oluşacak. İlk bölümü başlıktaki şekilde bir Rails uygulamasını docker ile ayağa kaldırmak, ikinci bölüm redis ve sidekiq servislerini hazırlamak ve üçüncü bölüm bu uygulamayı docker ile deploy etmek.

Çokça detaylı sıkıcı bir kaynak olmayacak. Çünkü:

1. [Docker nedir, ne işe yarar, nerede nasıl kullanılır?](https://www.gokhansengun.com/docker-nedir-nasil-calisir-nerede-kullanilir/)
2. [Yeni bir docker image'ı nasıl hazırlanır?](https://www.gokhansengun.com/docker-yeni-image-hazirlama/)
3. [Docker-compose hangi amaçlarla kullanılır?](https://www.gokhansengun.com/docker-compose-nasil-kullanilir/)

3 başlıkta [Gökhan hoca](https://twitter.com/gokhansengun) her şeyi detaylı anlatmış.

Rails uygulamam bir postgresql veritabanına sahip. Session, cache ve queue için redis, background jobları için sidekiq kullanıyorum.

İlk iş bir Dockerfile oluşturmak oldu, ilk denemelerimde slim ve ya slim-stretch image'larını kullanmıştım. Sonra image'larımın ne kadar küçük olursa o kadar iyi olacağı kanaatine vardım ve alpine image'ları kullanmaya başladım.

### Dockerfile

Uygulamam ruby 2.4.3 ile yazıldı. O yüzden Docker image'ım `ruby:2.4.3-alpine3.7` olacak. Uygulamamın root dizininde aşağıdaki gibi bir dosya oluşturuyorum.

./Dockerfile

```dockerfile
FROM ruby:2.4.3-alpine3.7

LABEL maintainer="Murat Bastas <muratbsts@gmail.com>"

ENV RAILS_ENV=development

EXPOSE 3000 4000
WORKDIR /project

COPY Gemfile Gemfile.lock /project/

COPY . /project

CMD ["bundle", "exec", "rails", "server", "-p 3000", "-b 0.0.0.0"]
```

Dockerfile yazıldıktan sonra proje dizinimde `docker build .` komutunu çalıştırınca docker image'ının ilk halini oluşturdum. Bunu aşağıdaki komutla görebiliriz.

```zsh
~/dev/rails-docker
$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED              SIZE
<none>              <none>              ca443db6f961        About a minute ago   60.9MB
```

Fakat bununla bitmedi. Bize postgresql ve redis lazım, ve hatta sidekiq çalıştıran bir servis lazım. Unutmadan nodejs lazım assetlerimizi compile edebilmek için. Imagemagick lazım resimler upload edip onları boyutlandıracaksak. Veritabanımıza bağlanabilmek için `libpq` lazım, `postgresql-client` lazım. Her neyse, Dockerfile içindeki her RUN, CMD, EXEC gibi komut bir container katmanı oluşturuyormuş. Bu yüzden tek komutta aşağıdaki gibi ihtiyacımız olan tüm paketleri kurabiliriz.

```Dockerfile
...

WORKDIR /project
...

RUN apk -U upgrade \
 && apk add -t build-dependencies \
    build-base \
    libressl \
    libtool \
    postgresql-dev \
 && apk add \
    ca-certificates \
    file \
    git \
    imagemagick \
    libpq \
    nodejs \
    nodejs-npm \
    tzdata \
 && update-ca-certificates \
 && cd /project \
 && rm -rf /tmp/* /var/cache/apk/*

...
```

Tekrar build diyerek bu paketlerin kurulduğunu görebiliriz, ama son halini alana kadar tekrar build demeyeceğim.

Şimdi bize bir de `bundle install` yaparak gemlerimizi kurmak düşüyor.

```zsh
...

COPY Gemfile Gemfile.lock /project/

RUN bundle install -j $(getconf _NPROCESSORS_ONLN) --no-deployment

...
```

Yukarıdaki şekilde `bundle install` ile Gemfile'ımızdaki gem'leri kurduk. Uygulamam birden fazla servis kullanacağı için(`postgresql`, `redis`, `sidekiq`, `web`) Dockerfile'ın olduğu dizinde `docker-compose.yml` oluşturacağım.

### docker-compose.yml

En basit haliyle postgresql ve web uygulamamız için iki tane servisi olan bir docker-compose.yml dosyası hazırladım.

```yaml
version: '3'

services:
  postgres:
    image: postgres:10-alpine
    restart: always
    environment:
      - POSTGRES_USER=project_dbuser
      - POSTGRES_DB=project_db
    ports:
      - "5432:5432"

  app:
    build: .
    command: bin/bundle exec rails s -p 3000 -b '0.0.0.0'
    env_file: .env.production
    depends_on:
      - postgres
    ports:
      - "3000:3000"
    volumes:
      - .:/project
```

Yukarıdaki `app` bloğunda `env_file` diye belirttiğim environment değerlerini saklayacak bir production(`.env.production`) environment'ı oluşturdum. Uygulamamızın ihtiyaç duyduğu diğer

```env
PORT=3000
RAILS_ENV=production
RAILS_SERVE_STATIC_FILES=true

DATABASE_URL=postgres://project_dbuser@postgres/project_db
```

Hemen peşine:

```zsh
~/dev/rails-docker
$ docker-compose build
```

diyerek servislerimin image'larını da build ettim.

Bundan sonra `docker ..` komutları yerine `docker-compose ..` kullanabiliriz. Mesela:

```zsh
~/dev/rails-docker
$ docker-compose run app bin/rails generate controller home index
Starting railsdocker_postgres_1 ... done
      create  app/controllers/home_controller.rb
       route  get 'home/index'
      invoke  erb
      create    app/views/home
      create    app/views/home/index.html.erb
      invoke  helper
      create    app/helpers/home_helper.rb
      invoke  assets
      invoke    js
      create      app/assets/javascripts/home.js
      invoke    css
      create      app/assets/stylesheets/home.css
```

Temel olarak sık kullanacağım komutlar:

```zsh
~/dev/rails-docker
$ docker-compose up [SERVIS_ADI (ihtiyac var ise)] [OPTIONS (belki -d, daemon olarak çalışmasını istersek)]
--------

~/dev/rails-docker
$ docker-compose down
--------

~/dev/rails-docker
$ docker-compose logs -f SERVIS_ADI
--------

~/dev/rails-docker
$ docker-compose run [SERVIS_ADI] ...
```

olacak. Ve artık:

```zsh
~/dev/rails-docker
$ docker-compose up
```

dediğimizde localhost:3000 adresinde uygulamamız production environment'ı ile ayağa kalkacak.

Bir sonraki bölümde görüşmek üzere.
