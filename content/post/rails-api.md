---
title: Rails API-only uygulama ve token authentication
date: 2017-03-30T19:19:22+30:00
authors:
  - Murat Bastas
---

Rails ile sadece API --**API-only**-- uygulama yazmak istiyorsanız terminalde `rails g proje --api` ile projeyi oluşturmalısınız. (bkz: [edgeguides](http://edgeguides.rubyonrails.org/api_app.html))

_Eminim rails ile uygulama geliştiren hemen hemen herkes kullanıcı işlemleri için [devise](https://github.com/plataformatec/devise) kullanıyordur. Devise daha ben rails ile geliştirmeye başlamadan çok evvel TokenAuthenticatable metoduna sahipmiş fakat [bu metod kaldırılmış.](https://github.com/plataformatec/devise/wiki/How-To:-Simple-Token-Authentication-Example)_

_[Bu wiki sayfası](https://github.com/plataformatec/devise/wiki/How-To:-Simple-Token-Authentication-Example)nda
token ile oturum açmak için birkaç yol verilmiş. Ama **API-only** uygulamada her denememde CookieStore ve SessionStore hatası aldım bu gemler ile. Zaten tüm örnek uygulamalarda ve readme dökümanlarında **API-only** değil **`ActionController::Base`** den kullanılmış. Bir süre araştırdıktan sonra kendi tokenlerimi oluşturup uygulamaya koymaya karar verecektim ki [tiddle](https://github.com/adamniedzielski/tiddle/) gemini buldum. Tam aklımda kurduğum gibi çalışıyor. Şimdi bu gem ile API uygulamamı nasıl hazırladığımı anlatacağım._

## API-only uygulama oluşturmak

```zsh
rails new api-only-app -T -B -d postgresql --api
```

Bu komut ile oluşturduğunuz rails uygulamasınında **ApplicationController** **`ActionController::Base`** sınıfından değil **`ActionController::API`** sınıfından extend edilmiş olacak ve içinde csrf\_token kontrolü yapan `protect_from_forgery with: :exception` şeklindeki middleware olmayacak. `assets` dizini hiç olmayacak.
`config/application.rb` içinde `config.api_only = true` şeklinde bir set göreceksiniz. `Gemfile` içinde sass, coffee, uglifier, jquery vb assetler ile alakalı hiç bir gem olmayacak.

Öncelikle gerekli gemleri kuralım. API uygulamanıza farklı domainlerden erişim olacak ise rack-cors kuracağız. Authentication için devise + tiddle kullanacağız. Hazırsak başlayayım.
Gemfile'ı aşağıdaki gibi düzenleyelim.

```zsh
# Gemfile
gem 'rack-cors', :require => 'rack/cors'

gem 'devise'
gem 'tiddle'
```

Ve şimdi `bundle install`.

Gemler kurulduktan sonra...

### Cors'u ayarlayalım

`config/initializers/cors.rb` adında bir initializer oluşturup içeriğini aşağıdaki şekilde düzenleyelim.

```zsh
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins '*'

    resource '*',
      headers: :any,
      expose: ['X-USER-TOKEN', 'X-USER-EMAIL'],
      methods: [:get, :post, :put, :delete, :options]
  end
end
```

Bu kadar.. 😊

### Devise'ı hazırlayalım

```zsh
rails g devise:install
rails g devise User
```

API-only uygulamada tüm requestlerin sonucu json döneceği için devise'ı aşağıdaki şekilde yapılandırmalıyız.

```zsh
# config/initializers/devise.rb
Devise.setup do |config|
    config.navigational_formats = []
end
```

**Önemli**: Bazı kaynaklarda `config.navigational_formats: [:json]` tanımlanıyor. Ama boş array olarak tanımlanmazsa flash message hataları alıyoruz

TokenAuthenticatable için `user.rb` modeline aşağıdaki gibi `:token_authenticatable` ekleyelim.

```zsh
class User < ActiveRecord::Base
  devise :database_authenticatable, :registerable,
         :recoverable, :trackable, :validatable,
         :token_authenticatable
  # ...
end
```

Tiddle için **AuthenticationToken** modelini oluşturalım.

```zsh
rails g model AuthenticationToken body:string user:references last_used_at:datetime ip_address:string user_agent:string
```

Ve şimdi `rake db:migrate`.

### Bir model oluşturalım

Örnek için scaffold üzerinden ilerleyeceğim.

```zsh
rails g scaffold post title:string body:text --api
```

Ve şimdi `rake db:migrate`.

### Controller dosyalarını düzenleyelim

Scaffold ile oluşturduğumuz controller dosyasını **controllers/api/v1** içine taşıyalım.

```zsh
mv app/controllers/posts_controller.rb app/controllers/api/v1/posts_controller.rb
```

Taşıdıktan sonra controller adını da aşağıdaki gibi değiştirmeyi unutmayalım ve before_action callback ayarlayarak create, update, destroy metodların koruyalım.

```zsh
class Api::V1::PostsController < ApplicationController
    before_action :authenticate_user!, only: [:create, :update, :destroy]
    # ...
end
```

Devise metodlarının üzerine yazmak için **api/auth** içinde `sessions_controller.rb` ve `registrations_controller.rb` dosyalarını oluşturalım.

SessionsController da create metodunun üzerine yazarak authenticate den sonra bir token oluşturup döndürelim.

```zsh
# sessions_controller.rb

class Api::Auth::SessionsController < Devise::SessionsController

  def create
    user = warden.authenticate!(auth_options)
    token = Tiddle.create_and_return_token(user, request)
    render json: { authentication_token: token }
  end

  def destroy
    Tiddle.expire_token(current_user, request) if current_user
    render json: {}
  end

  private

    # bu metod before_destroy callback i olarak çalışıyor. biz bunun üzerine yazıyoruz.
    def verify_signed_out_user
    end

end
```

RegistrationsController da flash mesajları yok edelim.

```zsh
# registrations_controller.rb

class Api::Auth::RegistrationsController < Devise::RegistrationsController

  def create
    build_resource(sign_up_params)

    resource.save
    yield resource if block_given?
    if resource.persisted?
      if resource.active_for_authentication?
        # set_flash_message! :notice, :signed_up if is_navigational_format?
        sign_up(resource_name, resource)
        return render :json => {:success => true, :data => resource}
      else
        # set_flash_message! :notice, :"signed_up_but_#{resource.inactive_message}" if is_navigational_format?
        expire_data_after_sign_in!
        return render :json => {:success => true, :data => resource}
      end
    else
      clean_up_passwords resource
      return render :json => {:success => false, :error => resource.errors}
    end
  end

end
```

### Rotalarımızı ayarlayalım

Authentication için api/auth ve uygulama rotaları için /api/v1/controller şeklinde rota kullanacağım.

```zsh
Rails.application.routes.draw do

  devise_for :users,
              path: 'api/auth',
              defaults: { format: :json },
              controllers: {
                sessions: 'api/auth/sessions',
                registrations: 'api/auth/registrations'
              }

  namespace :api, defaults: { format: :json } do
    namespace :v1 do
      resources :posts
    end
  end

end
```

### Çalıştıralım

```zsh
rails s -p 3000
```

En basit haliyle bir API-only uygulama yazmış ve çalıştırmış olduk. Şimdi bu uygulamayı test edelim.

```zsh
# kayıt olmak için

curl -X POST http://localhost:3000/api/auth --data '{user:{email:'yahya@kemal.im',password:'12345678',password_confirmation:'12345678'}}'
# sonuç
{
  "success": true,
  "data": {
    "id": 6,
    "email": "yahya@kemal.im",
    "created_at": "2017-03-30T23:13:23.879Z",
    "updated_at": "2017-03-30T23:13:23.918Z"
  }
}

# giriş yapmak ve token almak için

curl -X POST http://localhost:3000/api/auth/sign_in --data '{user:{email:'yahya@kemal.im',password:'12345678'}}'
# sonuç
{
  "authentication_token": "yYZpAEa3kxid3E6ZLHQp"
}

# posts#index için

curl -X GET http://localhost:3000/api/v1/posts

# sonuç
{
  "success": true,
  "data": {
    "id": 1,
    "title": "Test 1",
    "created_at": "2017-03-30T23:13:23.879Z",
    "updated_at": "2017-03-30T23:13:23.918Z"
  }
}

# posts#create için

curl -X POST http://localhost:3000/api/v1/posts \
  --header 'x-user-email: yahya@kemal.im' \
  --header 'x-user-token: yYZpAEa3kxid3E6ZLHQp' \
  --data '{posts:{title:"Test 2"}}'

# sonuç
{
  "success": true,
  "data": {
    "id": 1,
    "title": "Test 2",
    "created_at": "2017-03-30T23:13:23.879Z",
    "updated_at": "2017-03-30T23:13:23.918Z"
  }
}
```

Evet diğer korunan requestleri de aynı şekilde x-user-email ve x-user-token üst bilgilerini(header) göndererek güncelleyebilirsiniz.

Birilerine faydalı olduğunu umarak, esen kalın 🙏🏻
