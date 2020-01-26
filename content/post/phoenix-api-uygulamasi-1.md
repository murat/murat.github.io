---
title: "Phoenix API uygulaması (part 1)"
date: 2019-04-13T14:16:34+30:00
authors:
  - Murat Bastas
---

Merhabalar, uzun bir süredir yan projelerde kullanarak öğrenmeye çalıştığım Elixir dili için bir rehber niteliğinde notlar paylaşacağım yazı serime hoş geldiniz. Bu yazıya sıklıkla Ruby on Rails ile karşlılaştırılmaları yapılan Phoenix web framework ile API-only bir uygulama yazma rehberi olarak bakabilirsiniz. Hazırsam başlayayım... :)

## Yeni bir Phoenix projesi oluşturma

Bu adımda Elixir, Erlang ve Phoenix framework'ün sisteminizde kurulu olduğunu varsayıyorum. Değilse eğer sizi [şuraya](/2019/04/13/hepsine-h%C3%BCkmedecek-tek-versiyon-y%C3%B6neticisi-asdf/) yönlendirebilirim.

Aşağıdaki komutu terminalimizde çalıştırarak bir phoenix projesi açalım.

```zsh
mix phx.new app --no-html --no-webpack
```

`--no-html` ve `--no-webpack` flagleri projemizin bir front-endi olmadığını belirtiyor, ve `phx.new` mix taskı da bunu anlayarak front-endsiz bir proje açıyor. Bu komut bulunduğunuz dizinde `./app` adında bir dizin açarak içine phoenix boilerplate'i yerleştirecektir. Peşine `mix deps.get` taskını çalıştırarak bağımlılıkları indirecek ve dizinin içine girecektir.

Şimdi bir API uygulamasının olmazsa olmazı olan auth endpointlerinden başlayabiliriz.

## Auth contexti ile User modeli

Öncelikle `Auth` contexti altında `User` data modelimizi oluşturalım.

```zsh
mix phx.gen.context Auth User users name:string email:string:unique password_digest:string confirmed_at:datetime
```

Phoenix bu `Auth` contexti altında `User` data modelimizi oluştururken `users` adında bir tablo oluşturmak için de gerekli migration dosyalarımızı `priv/repo/migrations` dizini altına oluşturdu. `password_digest` isimli `string` olan alanımız `bcrypt` ile şifrelemek için `mix.exs` dosyasında `deps` fonksiyonuna aşağıdaki hex paketini ekleyelim.

```ex
# Add bcrypt for pass hashing in mix.exs
{:bcrypt_elixir, "~>  2.0"}
```

`mix deps.get` komutunu çalıştırınca `bcrypt_elixir` paketi `hex.pm` den indirilerek yüklenecektir.

Bu arada testlerde performansı arttırmak amacı ile `config/test.exs` konfigürasyon dosyasına aşağidaki ayarları ekleyerek bcrypt güvenlik seviyesini düşürelim.

```ex
# config/test.exs
config :bcrypt_elixir, :log_rounds, 4
```

`lib/app/auth/user.ex` dosyasını açıp `password` adında sanal bir field oluşturalım.

```ex
field :password, :string, virtual: true
field :password_digest, :string
```

`virtual: true` bu field'ın veritabanında olmadığını belirtmektedir. Bu field'ı kullanıcı girdisi alacağız ve üzerinde validasyonlarımızı yapıp, testlerimizden geçer ise `password_digest` verisini bcrypt ile üreteceğiz.

Bunun için `changeset` fonksiyonunu aşağıdaki gibi değiştirelim.

```ex
def changeset(user, attrs) do
  user
  |> cast(attrs, ~w(name email password confirmed_at)a)
  |> validate_required(~w(name email password)a)
  # E-mail validasyonu
  |> unique_constraint(:email)
  |> validate_format(:email, ~r/([\w-\.]+)@((?:[\w]+\.)+)([a-zA-Z]{2,4})/)
  # Password validasyonu (min 6 karakter ve confirmation ister)
  |> validate_length(:password, min: 6)
  |> validate_confirmation(:password)
  # Changeset'e password_digest ekle
  |> put_password_digest()
end

defp put_password_digest(
    %Ecto.Changeset{valid?: true, changes: %{password: password}} = changeset
  ) do
  change(changeset, password_digest: Bcrypt.hash_pwd_salt(password))
end

defp put_password_digest(changeset) do
  changeset
end
```

Yukarıda yorum satırlarında bahsettiğim gibi validasyonlarımız gerçekleşecek ve sıkıntısız changeset geldi ise bcrypt ile şifrelenmiş bir `password_digest` değeri oluşacaktır.

Otomatik oluşturulan auth testleri bizim eklediklerimiz ile bozulduğu için orayı düzeltsek iyi olur. `*_attrs` değerlerine bakınız...

```ex
# test/app/auth_test.exs

defmodule App.AuthTest do
  use App.DataCase

  alias App.Auth

  describe "users" do
    alias App.Auth.User

    @valid_attrs %{
      confirmed_at: ~N[2010-04-17 14:00:00],
      email: "email@email.com",
      locked_at: ~N[2010-04-17 14:00:00],
      name: "name",
      password: "password",
      password_confirmation: "password"
    }
    @update_attrs %{
      confirmed_at: ~N[2011-05-18 15:01:01],
      email: "updated@email.com",
      locked_at: ~N[2011-05-18 15:01:01],
      name: "updated name",
      password: "updated.password",
      password_confirmation: "updated.password"
    }
    @invalid_attrs %{
      confirmed_at: nil,
      email: nil,
      locked_at: nil,
      name: nil,
      password: nil,
      password_confirmation: false
    }

    def user_fixture(attrs \\ %{}) do
      {:ok, user} =
        attrs
        |> Enum.into(@valid_attrs)
        |> Auth.create_user()

      user
    end

    test "list_users/0 returns all users" do
      user = user_fixture()
      assert Auth.list_users() == [%User{user | password: nil}]
    end

    test "get_user!/1 returns the user with given id" do
      user = user_fixture()
      assert Auth.get_user!(user.id) == %User{user | password: nil}
    end

    test "create_user/1 with valid data creates a user" do
      assert {:ok, %User{} = user} = Auth.create_user(@valid_attrs)
      assert user.confirmed_at == ~N[2010-04-17 14:00:00]
      assert user.email == "email@email.com"
      assert user.locked_at == ~N[2010-04-17 14:00:00]
      assert user.name == "name"
      assert Bcrypt.verify_pass("password", user.password_digest)
    end

    test "create_user/1 with invalid data returns error changeset" do
      assert {:error, %Ecto.Changeset{}} = Auth.create_user(@invalid_attrs)
    end

    test "update_user/2 with valid data updates the user" do
      user = user_fixture()
      assert {:ok, %User{} = user} = Auth.update_user(user, @update_attrs)
      assert user.confirmed_at == ~N[2011-05-18 15:01:01]
      assert user.email == "updated@email.com"
      assert user.name == "updated name"
      assert Bcrypt.verify_pass("updated.password", user.password_digest)
    end

    test "update_user/2 with invalid data returns error changeset" do
      user = user_fixture()
      assert {:error, %Ecto.Changeset{}} = Auth.update_user(user, @invalid_attrs)
      assert %User{user | password: nil} == Auth.get_user!(user.id)
    end

    test "delete_user/1 deletes the user" do
      user = user_fixture()
      assert {:ok, %User{}} = Auth.delete_user(user)
      assert_raise Ecto.NoResultsError, fn -> Auth.get_user!(user.id) end
    end

    test "change_user/1 returns a user changeset" do
      user = user_fixture()
      assert %Ecto.Changeset{} = Auth.change_user(user)
    end
  end
end
```

## CORS (Cross-Origin Resource Sharing)

API'ımıza hangi originlerden ve hangi methodlarla erişilebileceğini belirtmek ve ya kısıtlamak için `mix.exs`e `cors_plug` paketini ekliyoruz.

```ex
{:cors_plug, "~> 2.0"}
```

Ve ardından `mix deps.get` ile paketi indirirken `config/dev.exs` ve `config/prod.exs` dosyalarında aşağıdaki tanımlamaları yapıyoruz.

```
# config/dev.exs

config :cors_plug,
  origin: ~r/localhost/,
  max_age: 86400, # 24 hours
  methods: ["GET", "POST", "PUT", "PATCH", "DELETE"]

# config/prod.exs

config :cors_plug,
  origin: ["https://domain.com", "https://example.com"],
  max_age: 86400, # 24 hours
  methods: ["GET", "POST", "PUT", "PATCH", "DELETE"]
```

## İlk endpointimiz

Phoenix de aynı Rails gibi güzel generatorlere([mix tasks](https://hexdocs.pm/phoenix/phoenix_mix_tasks.html)) sahip. Json endpointleri hazırlamak için aşağıdaki mix taskını kullanabiliriz.

```zsh
mix phx.gen.json Auth User users name:string email:string password:string --no-context --no-schema

# it asks for proceed that with existed Auth context, you can say [Y]
    * creating lib/app_web/controllers/user_controller.ex
    * creating lib/app_web/views/user_view.ex
    * creating test/app_web/controllers/user_controller_test.exs
    * creating lib/app_web/views/changeset_view.ex
    * creating lib/app_web/controllers/fallback_controller.ex

    Add the resource to your :api scope in lib/app_web/router.ex:

        resources "/users", UserController, except: [:new, :edit]
```

`router.ex` dosyamıza rotamızı ekleyelim.

```ex
scope "/api", AppWeb do
  pipe_through :api
  ...
  scope "/v1", V1 do
    resources "/users", UserController, except: [:new, :edit]
  end
end
```

Tarayıcımızdan [`http://localhost:4000/api/v1/users`](http://localhost:4000/api/v1/users) adresine gidersek yeni rotamızı görebiliriz.

Tabi user_controller_test dosyamızı da testleri geçecek şekilde güncellemeyi de ihmal etmeyelim. Özellikle password fieldını gördüğümüz her yerden kaldıralım.

`lib/app_web/views/user_view.ex` dosyasında da `password` ü kaldırmayı unutmayalım.

Şimdi [şuradan devam](/2019/04/14/phoenix-api-uygulaması-part-2/) edebilirsiniz...
