---
title: "Phoenix API uygulaması (part 3)"
date: 2019-05-12T12:15:12+30:00
authors:
  - Murat Bastas
---

İkinci bölümünü [neredeyse 1 ay önce yayınladığım phoenix framework API uygulaması](/2019/04/14/phoenix-api-uygulaması-part-2/) serisinin üçüncü yazısıdır.

İkinci bölümde oauth ile token alıp rotalarımızı bu token ile korumayı öğrenmiştik. Bu bölümde de kayıt olma esnasında e-posta göndererek kullanıcıların e-posta adreslerini doğrulama işlemlerini yapacağız.

Yaptığım araştırmalar sonucunda Phoenix'e e-posta gönderme yetenekleri eklemek için en kullanışlı hex paketinin [bamboo](https://github.com/thoughtbot/bamboo) olduğunu düşünüyorum. Bu yüzden bamboo ile development ortamında Rails'te kullandığımız letter_opener gibi çalışan local adaptörü, production ortamında ise SMTP adaptörü kullanacağız.

```ex
  defp deps do
    [
      ...
      {:bamboo, "~> 1.2"},
      {:bamboo_smtp, "~> 1.6.0"},
      ...
    ]
  end
```

`mix deps.get` komutu ile paketleri çektikten sonra aşağıdaki config dosyaslarına aşağıdaki konfigürasyonları ekliyoruz.

```ex
# config/dev.exs
config :app, App.Mailer, adapter: Bamboo.LocalAdapter

# config/test.exs
config :app, App.Mailer, adapter: Bamboo.TestAdapter

# config/prod.exs
config :app, App.Mailer, adapter: Bamboo.SMTPAdapter,
  server: System.get_env("SMTP_HOSTNAME"),
  hostname: System.get_env("SMTP_HOSTNAME"),
  port: System.get_env("SMTP_PORT"),
  username: System.get_env("SMTP_USERNAME"),
  password: System.get_env("SMTP_PASSWORD"),
  tls: :if_available,
  retries: 1
```

Sengrid, mandrill, mailgun vb başka adaptörleri kullanmak isterseniz bamboo'nun readme dosyasında [şuraya](https://github.com/thoughtbot/bamboo#available-adapters) göz atabilirsiniz.

## Mailer oluşturma

`lib/app_web` dizininde bir `email` dizini oluşturup içine `user_mailer.ex` adıyla bir modül oluşturalım ve içeriğini aşağıdaki gibi dolduralım.

```ex
defmodule AppWeb.UserMailer do
  use Bamboo.Phoenix, view: AppWeb.EmailView

  def confirmation_email(user, confirmation_url) do
    base_email()
    |> to(user.email)
    |> subject("Welcome! Please confirm your account.")
    |> assign(:user, user)
    |> assign(:confirmation_url, confirmation_url)
    |> render(:confirmation)
  end

  defp base_email do
    new_email()
    |> from("App <noreply@app.com>")
    |> put_html_layout({AppWeb.LayoutView, "email.html"})
  end
end
```

UserMailer'in EmailView kullanacağını ve `base_email/0` fonksiyonu ile de defaılt mail ayarlarını belirttik. `base_email/0` fonksiyonu `new_email/0` fonksiyonuna `from/2` ve `put_html_layout/2` fonksiyonlarını zincirleyerek bir e-mail döndürüyor ve `confirmation_email/2` fonksiyonu da bu e-maile gerekli assignment'ları yaparak `:confirmation` templatini veriyor.

Gerekli view'ları ve template dosyalarını oluşturalım.

`lib/app_web/templates/layout/email.html.eex` dosyasına html formatlı e-mail içeriğini hazırlıyoruz.

```html
<html>
  <head>
    <link
      rel="stylesheet"
      href='<%= Routes.static_url(AppWeb.Endpoint, "/css/email.css") %>'
    />
  </head>
  <body>
    <%= render @view_module, @view_template, assigns %>
  </body>
</html>
```

`<%= render @view_module, @view_template, assigns %>` satırı ile asıl göndermek istediğimiz e-mailin içeriğini bu template'in içine çağırıyoruz. O da mesela;

```html
<h1>Welcome <%= @user.name %></h1>

<p>
  Your confirmation url is
  <a href="<%= @confirmation_url %>"><%= @confirmation_url %></a>
</p>
```

Aynı şekilde düz metin halinde e-mail göndermek isterseniz `.html.eex`lerin yanına `.text.eex` ekleyebilirsiniz. Mesela `layout/email.text.eex` aşağıdaki şekilde olabilir;

```plain
<%= render @view_module, @view_template, assigns %>
```

`email/confirmation.text.eex` ise aşağıdaki gibi.

```plain
Welcome <%= @user.name %>

Your confirmation url is : <%= @confirmation_url %>
```

View modüllerini de `lib/app_web/views/email_view.ex` dosyası açıp içini aşağıdaki gibi bırakmak yeterli olacaktır.

```ex
defmodule AppWeb.EmailView do
  use AppWeb, :view
end
```

LayoutView zaten hali hazırda vardı. Yoksa aynı EmailView gibi oluşturabilirsiniz.

## Development ortamı

LocalAdapter bize development ortamında gönderdiğimiz maillere bir http rotası ile inbox içinde görme imkanı tanıyor. Bunun için öncelikle router'a aşağıdaki satırları eklememiz gerekiyor.

```ex
defmodule AppWeb.Router do
  use AppWeb, :router
  ...
  if Mix.env() == :dev do
    forward "/inbox", Bamboo.SentEmailViewerPlug
  end
end
```

Şimdi browserdan phoenix server çalışırken [http://localhost:4000/inbox](http://localhost:4000/inbox) adresine gidersek inbox göreceğiz.

## E-posta doğrulama ekleyelim

Önceki yazıları takip ettiyseniz, `register_controller` vardı yazdığımız. Orada `create/2` fonksiyonu içinde gerekli validasyonları yaparak kullanıcı oluşturmuştuk. Şimdi kullanıcı başarıyla oluşturulduğunda e-posta göndermek için `create/2` fonksiyonu aşağıdaki hale gelecek.

```ex
  ...
  def create(conn, %{"user" => user_params}) do
    case Auth.create_user(user_params) do
      {:ok, %User{} = user} ->
        AppWeb.UserMailer.confirmation_email(user)
        |> App.Mailer.deliver_now()

        conn
        |> put_status(:created)
        |> put_view(AppWeb.AuthView)
        |> render("user.json", user: user)

      {:error, changeset} ->
        conn
        |> put_status(:unprocessable_entity)
        |> put_view(AppWeb.ChangesetView)
        |> render("error.json", changeset: changeset)
    end
  end
  ...
```

Şimdi konfirmasyon işlemi yapmamız için bir token oluşturup e-posta içine doğrulama bağlantısı vermemiz gerekiyor. Bunun için ben Erlang'ın `:crypto` modülünden `.hash/2` fonksiyonunu kullanmayı tercih ediyorum. Kayıt olma aşamasında bir benzersiz anahtar üretip veritabanımıza ekleyeceğiz, ve bu anahtar ile `/auth/confirm?confirmation-token=TOKEN` şeklinde bir rotaya geleceğiz, burada gelen token ile kullanıcıyı bulup, varsa `confirmed_at` alanını dolduracağız. Mükemmel güvenli yolu bu olmasa da şimdilik öğrenmeye çalıştığımız şeye hizmet ediyor. Aşağıdaki komutu iex konsolunda deneyelim.

```ex
iex > :crypto.hash(:sha256, [1, "muratbsts@gmail.com", :crypto.strong_rand_bytes(12)]) |> Base.encode16 |> String.downcase
```

Önce users tablomuza gerekli alanları ekleyelim.

```zsh
mix ecto.gen.migration add_confirmation_to_users
```

```ex
# priv/repo/migrations/... add_confirmation_to_users.exs
  ...
  alter table(:users) do
    add :confirmation_token, :string

    add :confirmed_at, :utc_datetime # eger onceki bolumlerde eklemediyseniz
  end
```

```zsh
mix ecto.migrate
```

Şimdi register_controller dan `create/2` fonksiyonuna giderek mail gönderme işleminin hemen üstüne aşağıdaki kodu ekleyelim.

```ex
  ...
  def create(conn, %{"user" => user_params}) do
    case Auth.create_user(user_params) do
      {:ok, %User{} = user} ->

        confirmation_token =
          :crypto.hash(:sha256, [user.id, user.email, :crypto.strong_rand_bytes(12)])
          |> Base.encode16
          |> String.downcase

        Auth.update_user(user, %{"confirmation_token" => confirmation_token})

        AppWeb.UserMailer.confirmation_email(
          user,
          Routes.register_path(conn, :confirm, user.confirmation_token)
        )
        |> App.Mailer.deliver_now()
  ...
```

## Confirm işlemi

Register controller'da aşağıdaki gibi `confirm/2` fonksiyonu oluşturalım

```ex
  ...
  def confirm(conn, %{"token" => token}) do
    case Repo.one!(from(u in User, where: u.confirmation_token == ^token, limit: 1)) do
      nil ->
        conn
        |> put_status(:not_found)
        |> render("error.json")

      user ->
        if is_nil(user.confirmed_at) do
          case Auth.update_user(user, %{"confirmed_at" => DateTime.utc_now()}) do
            {:ok, user} ->
              conn
              |> put_status(:accepted)
              |> put_view(DataxWeb.AuthView)
              |> render("user.json", user: user)

            {:error, changeset} ->
              conn
              |> put_status(:unprocessable_entity)
              |> render("error.json", changeset: changeset)
          end
        else
          conn
          |> put_status(:accepted)
        end
    end
  end
```

Router'a da register resources rotasının hemen üstüne aşağıdaki rotayı eklediğimizde her şey tamamlanmış olacak.

```ex
  ...
  scope "/auth", Auth do
    get "/confirm/:token", RegisterController, :confirm
    resources "/sign_up", RegisterController, only: [:create]
    ...
  end
  ...
```

## İlk mailimizi gönderelim

`iex -S mix phx.server` ile development ortamında projemizi çalıştırıp konsola aşağıdaki satırı yapıştırdığımızda bir e-mail gönderilecektir.

```ex
iex > user = App.Auth.get_user!(1)
iex > token = :crypto.hash(:sha256, [user.id, user.email, :crypto.strong_rand_bytes(12)]) |> Base.encode16 |> String.downcase
iex > App.Auth.update_user(user, %{"confirmation_token" => token, "confirmed_at" => nil})


iex > conn = %Plug.Conn{private: %{phoenix_endpoint: AppWeb.Endpoint}}
iex > url = AppWeb.Router.Helpers.register_url(conn, :confirm, token)
iex > AppWeb.UserMailer.confirmation_email(user, url) |> App.Mailer.deliver_now()
```

Umarım faydalı bir yazı olmuştur, bir sonraki bölümde API uygulamamıza swagger ile dökümantasyon ekleyeceğiz. Ve bu seriyi bitireceğiz. Bu seri bitince başka seriler başlayacak. Esen kalın.

Şimdi [şuradan devam](/2019/05/19/phoenix-api-uygulamas%C4%B1-part-4/) edebilirsiniz...
