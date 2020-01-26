---
title: "Phoenix API uygulaması (part 4)"
date: 2019-05-19T13:37:02+30:00
authors:
  - Murat Bastas
---

Bu dördüncü yazıya kadar okuyan her elixir meraklasının api endpointleri nasıl hazırlayacağını, yetkilendirme ve giriş yapmayı nasıl halledeceğini öğrendiğini varsayıyorum. Sadece bu kadarı bir uygulama geliştirmeye yetmeyecektir elbet ama eksikleri ilgililer tamamlayacaktır diye umuyorum. Bu serinin son bölümü bir api'ın olmazsa olmazı, dökümanyasyonu üzerine olacak. Postman gibi araçlar kullanarak collection'lardan dökümantasyon üretmek mümkün tabii ki fakat yaygın olarak kullanılan swagger aracının phoenix projemize nasıl ekleneceğini anlatacağım.

Önce `mix.exs` e gerekli paketleri ekleyelim.

```ex
  ...
  defp deps do
    ...
    {:ex_json_schema, "~> 0.5"},
    {:phoenix_swagger, "~> 0.8"}
  end
  ...
```

Ve `mix deps.get` diyerek indirelim. Uygulamamız her derlendiğinde swagger dosyalarımızın da derlenmesi için `compilers` a swagger ekleyelim.

```ex
  # mix.exs
  ...
  def project do
    [
      ...
      compilers: [:phoenix, :gettext, :phoenix_swagger] ++ Mix.compilers(),
      ...
    ]
  end
  ...
```

`config/config.exs` dosyamızda da gerekli konfigürasyonları yapalım.

```ex
...
config :app, :phoenix_swagger,
  swagger_files: %{
    "priv/static/swagger.json" => [
      router: AppWeb.Router,
      endpoint: AppWeb.Endpoint
    ]
  }
```

`router.exs` dosyasına api dökümantasyon rotamızı ve swagger info fonksiyonunu verelim.

```ex
defmodule AppWeb.Router do
  use AppWeb, :router
  ...
  scope "/api/docs" do
    forward "/", PhoenixSwagger.Plug.SwaggerUI,
      otp_app: :app,
      swagger_file: "swagger.json"
  end

  def swagger_info do
    %{
      info: %{
        version: "0.1.0",
        title: "App"
      }
    }
  end
  ...
end
```

Şimdi aşağıdaki mix task'ı ile swagger.json dosyasını oluşturabilirsiniz.

```zsh
mix phx.swagger.generate
```

Phoenix server çalıştırdığınızda boş bir swagger dökümantasyonu göreeksiniz. Dökümanımızı doldurmak için endpointlerimize swagger için bir şeyler ekleyeceğiz.

`localhost:4000/api/v1/users` endpointi için döküman ekleyelim. `user_controller.ex` dosyasını aşağıdaki ekleri yapalım ve tekrar swagger dosyamızı oluşturalım.

```ex
defmodule AppWeb.V1.UserController do
  use AppWeb, :controller
  use PhoenixSwagger


  def swagger_definitions do
    %{
      User:
        swagger_schema do
          title("User")
          description("A user of the app")

          properties do
            id(:integer, "User ID")
            name(:string, "User name", required: true)
            email(:string, "Email address", format: :email, required: true)
            inserted_at(:string, "Creation timestamp", format: :datetime)
            updated_at(:string, "Update timestamp", format: :datetime)
          end

          example(%{
            id: 123,
            name: "Joe",
            email: "joe@gmail.com",
            inserted_at: "2019-04-03T21:16:09",
            updated_at: "2019-04-03T21:16:09"
          })
        end,
      UserRequest:
        swagger_schema do
          title("UserRequest")
          description("POST body for creating a user")
          property(:user, Schema.ref(:User), "The user details")
        end,
      UserResponse:
        swagger_schema do
          title("UserResponse")
          description("Response schema for single user")
          property(:data, Schema.ref(:User), "The user details")
        end,
      UsersResponse:
        swagger_schema do
          title("UsersReponse")
          description("Response schema for multiple users")
          property(:data, Schema.array(:User), "The users details")
        end
    }
  end

  swagger_path :index do
    get("/api/v1/users")
    summary("Get users")
    produces("application/json")
    tag("users")

    paging()

    response(200, "OK", Schema.ref(:UsersResponse))
    response(401, "Unauthorized")
  end

  swagger_path :show do
    get("/api/v1/users/:id")
    summary("Get an user")
    produces("application/json")
    tag("users")

    response(200, "OK", Schema.ref(:UserResponse))
    response(401, "Unauthorized")
  end

  swagger_path :create do
    post("/api/v1/users")
    summary("Create user")
    description("Creates a new user")
    consumes("application/json")
    produces("application/json")
    tag("users")

    parameter(:user, :body, Schema.ref(:UserRequest), "The user details",
      example: %{
        user: %{name: "Joe", email: "Joe1@mail.com"}
      }
    )

    response(200, "OK", Schema.ref(:UserResponse))
    response(401, "Unauthorized")
    response(403, "Forbidden")
  end
  ...
end
```

```zsh
mix phx.swagger.generate
```

Şimdi oluşturulan swagger dökümantasyonumuza browserdan bakabiliriz. [`localhost:4000/api/docs`](http://localhost:4000/api/docs) adresine sayfasını ziyaret edelim.

Aşağıdaki gibi bir ekran gördüyseniz her şey yolunda demektir.

{{< figure src="https://cdn-images-1.medium.com/max/1600/1*hN-SPAI29s5UJCizmBxyIw.png" >}}

Diğer tüm endpointleriniz için aynı yukarıdaki gibi schema_definitions ve response models eklerseniz `phx.swagger.generate` komutu bu arayüzü güncelleyecektir. Swagger dökümantasyonunuzu birden fazla dosyaya da bölebilirsiniz, örneğin oturum açma ve yetkilendirme dökümanlarını ayrı, kullanıcılarınız ile alakalı dökümanları ayrı, public olanları ayrı vb bölümlendirebilirsiniz. Bunun için swagger'in kendi dökümanını ve phoenix_swagger'in exdoc dökümanını biraz incelemeniz yeterli.

Bu phoenix API uygulaması serisinin son yazısıdır, önceki bölümlere aşağıdaki bağlantılardan erişebilirsiniz. Başka bir yazıda görüşmek üzere.

- 1 - [Phoenix proje oluşturma, kullanıcılar, ilk json endpoint](/2019/04/13/phoenix-api-uygulamas%C4%B1-part-1/)
- 2 - [Giriş yapma ve yetkilendirme, rotaları koruma](/2019/04/14/phoenix-api-uygulaması-part-2/)
- 3 - [E-posta göndererek kullanıcı teyit etme](/2019/05/12/phoenix-api-uygulamas%C4%B1-part-3/)
- 4 - (Şu an buradasınız :) Swagger ile döküman üretme
