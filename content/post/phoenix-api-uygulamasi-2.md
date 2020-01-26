---
title: "Phoenix API uygulaması (part 2)"
date: 2019-04-14T17:35:12+30:00
authors:
  - Murat Bastas
---

İlk halini [dün yayınladığım phoenix framework API uygulaması](/2019/04/13/phoenix-api-uygulamas%C4%B1-part-1/) serisinin ikinci yazısıdır.

Dün basitçe crud işlemleri yapan bir endpoint hazırlamıştık. Tabi ki sırada bir API ın olmazsa olmazı authentication işlemlerini nasıl yapacağımızdan bahsedeceğim.

Öncelikle ihtiyaç duyacağımız paketlerden bahsedelim. Ruby yazanlar ve Rails ile uygulama geliştirenler bilirler plataformatec'in geliştirdiği ve José Valim abimizin, yani Elixir dilinin yaratıcısının çok büyük katkıları olan [devise](https://github.com/plataformatec/devise) adında bir gem var ki authentication adına ne var ise her şeyi 3-5 generator komutu ile saniyeler içerisinde halledebiliyorsunuz. Elixir topluluğu için geliştirilen [coherence](https://github.com/smpallen99/coherence) adında bir devise alternatifi olsa da ben authentication işlemleri için yaygın olarak [guardian](https://github.com/ueberauth/guardian) paketini kullanmayı tercih ediyorum. Bu bölümde de guardian ile basit bir authentication nasıl yapılır bundan bahsedeceğim.

Paketleri deps fonksiyonuna ekleyerek başlayalım.

```ex
defp deps do
  # bcrypt eklemiştik ama altını tekrar çizelim...
  {:bcrypt_elixir, "~>  2.0"},
  # ...
  {:guardian, "~> 1.2"}
  # ...
end
```

`mix deps.get` diyerek guardian'ı projemize ekliyoruz. Dileyen [şuraya da](https://github.com/ueberauth/guardian/blob/master/guides/tutorial/start-tutorial.md) bakabilir.

Guardian bize JWT token üretme imkanı tanıyacak, bunun için ufak bir konfigürasyon yapmakta fayda var. `config/config.exs` dosyamıza aşağıdaki konfigürasyonu ekleyelim.

```ex
# ...
config :app, App.Auth.Guardian,
  issuer: "app",
  secret_key: System.get_env("GUARDIAN_SECRET_KEY")
# ...
```

**Not** `App.Auth.Guardian` modülünü birazdan oluşturacağız.

`router.ex` dosyamızda gerekli değişiklikleri yapalım...

```ex
pipeline :ensure_auth do
  plug App.Auth.Guardian.Pipeline
end

scope "/api", AppWeb do
  pipe_through :api

  scope "/auth", Auth do
    resources "/sign_up", RegisterController, only: [:create]
    resources "/sign_in", SessionController, only: [:create]
  end
end

scope "/api", AppWeb do
  pipe_through [:api, :ensure_auth]

  scope "/v1", V1 do
    resources "/users", UserController, except: [:new, :edit]
  end
end
```

Bir önceki yazıda, `resources "/users"` resource'unu sadece `api` pipeline'ından geçirmiştik, şimdi bu resource rotasını yine `/api` scope'u içinde kalarak `pipe_through [:api, :ensure_auth]` pipeline'larından geçireceğiz. `:ensure_auth` pipeline'ında `App.Auth.Guardian.Pipeline` adında bir plug'ın varlığından bahsettik. `lib/app/auth/guardian/pipeline.ex` path'inde aynı isimle bir module tanımlayarak içeriğini aşağıdaki gibi yapalım.

```ex
defmodule App.Auth.Guardian.Pipeline do
  use Guardian.Plug.Pipeline, otp_app: :app,
                              module: App.Auth.Guardian,
                              error_handler: App.Auth.Guardian.ErrorHandler

  # Requestlerde authorization header arayıp, token bir access token ise doğrula
  plug Guardian.Plug.VerifyHeader, claims: %{"typ" => "access"}
  plug Guardian.Plug.EnsureAuthenticated
end
```

Bu oluşturduğumuz modül `Guardian.Plug.Pipeline` modülünü konfigüre eder. Yukarıdaki modülde görüldüğü gibi iki modüle daha ihtiyacı var, birincisi `App.Auth.Guardian` ikincisi `App.Auth.Guardian.ErrorHandler`, bu modülleri de oluşturalım ve içlerini aşağıdaki gibi dolduralım.

```ex
# lib/app/auth/guardian.ex
defmodule App.Auth.Guardian do
  use Guardian, otp_app: :app

  alias App.Auth

  def subject_for_token(resource, _claims), do: {:ok, to_string(resource.id)}

  def resource_from_claims(claims) do
    Auth.get_user!(claims["sub"])
  end
end

# lib/app/auth/guardian/error_handler.ex
defmodule App.Auth.Guardian.ErrorHandler do
  import Plug.Conn

  @behaviour Guardian.Plug.ErrorHandler
  @impl Guardian.Plug.ErrorHandler
  def auth_error(conn, {type, _reason}, _opts) do
    body = Poison.encode!(%{meta: %{status: to_string(type)}})

    conn
    |> put_resp_content_type("application/json")
    |> send_resp(401, body)
  end
end
```

`App.Auth.Guardian` modülündeki `resource_from_claims/1` fonksiyonu `claims` map'i içinde `"sub"` key'i ile kullanıcımızın `id` sini tutacak. Bu id ile `Auth` modülünde `get_user!/1` fonksiyonunu çağırarak giriş yapmış kullanıcının veritabanı kaydını bulacağız. `ErrorHandler` modülümüz ise authentication hatalarını alıp 401 http statüsü ile cevap verecek.

Şimdi `SessionController` modülümüzü oluşturalım.

```ex
defmodule AppWeb.Auth.SessionController do
  use AppWeb, :controller
  alias App.Auth

  action_fallback AppWeb.FallbackController

  def create(conn, %{"user" => %{"email" => email, "password" => password}}) do
    case Auth.authenticate_user(email, password) do
      {:ok, token, _claims}->
        conn
        |> put_view(AppWeb.AuthView)
        |> render("token.json", token: token)

      {:error, message} ->
        conn
        |> put_status(:unauthorized)
        |> put_view(AppWeb.AuthView)
        |> render("error.json", %{message: to_string(message)})
    end
  end
end
```

`SessionController.create/2` fonksiyonu parametre olarka gelen email ve password ile `Auth.authenticate_user/2` fonksiyonunu çalıştırıp başarılı ve ya hatalı durumlara göre malum cevapları verecektir. Şimdi `Auth` modülünde `authenticate_user/2` fonksiyonumuzu yazmamız gerekiyor.

```ex
# lib/app/auth.ex
defmodule App.Auth do
  # ...

  def authenticate_user(email, password) do
    case Repo.one(from(u in User, where: u.email == ^email)) do
      nil ->
        {:error, :invalid_credentials}
      user ->
        if Bcrypt.verify_pass(password, user.password_digest) do
          App.Auth.Guardian.encode_and_sign(user)
        else
          {:error, :invalid_credentials}
        end
    end
  end
end
```

Bu fonksiyon ise verilen email ile veritabanında kullanıcımızı arar, bulursa ve `Bcrypt.verify_pass/2` fonksiyonu ile verilen password ile veritabanındaki hashlenmiş passwordü test eder, verilen bilgiler doğru ise `Guardian.encode_and_sign` fonksiyonu ile bir `JWT` token ile birlikte claims oluşturup döndürür, bulamazsa ve ya password yanlış ise `{:error, :invalid_credentials}` şeklinde bir tuple döndürür.

`SessionController`'a dönersek response için `AuthView` kullanmıştık. `AuthView` modülümüzü de aşağıdaki gibi oluşturalım.

```ex
# lib/app_web/views/auth_view.ex

defmodule AppWeb.AuthView do
  use AppWeb, :view

  def render("token.json", %{token: token} = _) do
    %{token: token}
  end

  def render("error.json", %{message: message}) do
    %{error: message}
  end
end
```

**Sonuçlar**

Şimdi [`localhost:4000/api/v1/users`](http://localhost:4000/api/v1/users) adresine get requesti attığımızda aşağıdaki gibi bir cevap almamız gerekiyor.

```zsh
http :4000/api/v1/users | jq
HTTP/1.1 401 Unauthorized
cache-control: max-age=0, private, must-revalidate
content-length: 37
content-type: application/json; charset=utf-8
date: Sun, 14 Apr 2019 19:03:02 GMT
server: Cowboy
x-request-id: FZVsiSTULGUhyEwAAA3E

{
  "meta": {
    "status": "Unauthenticated"
  }
}
```

Gördüğünüz gibi `:ensure_auth` pipeline'ı yetkisiz erişimi engelledi. Önce gidip bir access token almamız gerekiyor. Bunu da aşağıdaki gibi deneyelim.

```zsh
http --form POST :4000/api/auth/sign_in 'user[email]=muratbsts@gmail.com' 'user[password]=password'
HTTP/1.1 200 OK
cache-control: max-age=0, private, must-revalidate
content-length: 467
content-type: application/json; charset=utf-8
date: Sun, 14 Apr 2019 19:13:31 GMT
server: Cowboy
x-request-id: FZVtG8GKfJ2sj3oAABIh

{
    "token": "eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.....JWRohW7yVxs978V4SV87g2BUFBT1BlpYNtsSfiOxPvss1hOiuUyZ1ZqIYOwyJ0-DYSPs63oEPGdA..."
}
```

Ve bu token ile users endpointimize request yaparsak aşağıdaki gibi bir başarılı bir sonuç alacağız.

```zsh
http :4000/api/v1/users 'Authorization:Bearer eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.....JWRohW7yVxs978V4SV87g2BUFBT1BlpYNtsSfiOxPvss1hOiuUyZ1ZqIYOwyJ0-DYSPs63oEPGdA...'
HTTP/1.1 200 OK
cache-control: max-age=0, private, must-revalidate
content-length: 1536
content-type: application/json; charset=utf-8
date: Sun, 14 Apr 2019 19:15:32 GMT
server: Cowboy
x-request-id: FZVtN9wwpCWV8KcAABJh

{
  "data": [
    {
      "email": "muratbsts@gmail.com",
      "id": 1,
      "inserted_at": "2019-04-03T21:16:09",
      "name": "murat",
      "updated_at": "2019-04-03T21:16:09"
    },
    ....
  }
]
```

Ve böylece ikinci bölümün de sonuna geldik. Bir sonraki bölümde kaydolan kullanıcılara email göndererek doğrulama yapacağız...

Şimdi [şuradan devam](/2019/05/12/phoenix-api-uygulamas%C4%B1-part-3/) edebilirsiniz...
