---
title: "Elixir OptionParser modülü ile CLI uygulaması"
date: 2019-05-23T22:12:05+30:00
authors:
  - Murat Bastas
---

Merhabalar, bu yazımda elixir ile yazacağımız komut satırı uygulamalarında dışarıdan verilen argümanları nasıl parse edebileceğimizden bahsedeceğim. Neredeyse her komut satırı uygulaması yazabildiğiniz dilde olduğu gibi elixir'de de argümanları ayrıştırmak için [`OptionParser`](https://hexdocs.pm/elixir/OptionParser.html) modülü ve [hex.pm](https://hex.pm)'den indirerek kullanabileceğimiz [`optimus`](https://github.com/savonarola/optimus) gibi kütüphaneler var. Birlikte `OptionParser` ile örnek bir uygulama yapacağız. Bu uygulama belirlenen sıradaki fibonacci sayısını istenen renkte ekrana yazdıracak.

```zsh
mix new fib
```

Diyerek başlayalım ve `lib/fib` dizininde `cli.ex` adında bir modül oluşturalım. İçinde `main/1` şeklinde bir fonksiyon olsun.

```ex
defmodule Fib.CLI do
  @moduledoc false

  def main(args \\ []) do
    IO.inspect(args)
  end
end
```

Çalışan bir dosya üretmek için `mix.exs` dosyamızda `escript` tanımlaması yapmalıyız.

```zsh
defmodule Fib.MixProject do
  use Mix.Project

  def project do
    [
      app: :fib,
      version: "0.1.0",
      elixir: "~> 1.8",
      start_permanent: Mix.env() == :prod,
      escript: [main_module: Fib.CLI],
      deps: deps()
    ]
  end
  ...
```

`escript` tanımlamasını yaptıktan sonra `mix escript.build` task'ını çalıştırdığımızda `fib` adında bir çalıştırabilir dosya üretecektir.

```zsh
./fib
[]

./fib --help
["--help"]
```

Bu argümanları ayrıştırmak için `OptionParser` modülünü kullanacağız. Dökümantasyonda gördüğünüz üzere çok basit.

```ex
def main(args \\ []) do
  {valid_opts, remains, invalid_opts} =
    args
    |> OptionParser.parse(
      switches: [help: :boolean, order: :integer, color: :string],
      aliases: [h: :help, n: :order, c: :color]
    )

  IO.inspect(valid_opts)
  IO.inspect(remains)
  IO.inspect(invalid_opts)
end
```

Şimdi `mix escript.build && ./fib --help -n 10 -c red`  şeklinde çalıştırdımızda `[help: true, order: 10, color: "red"]` şeklinde bir keyword list ekrana basacaktır. `switches` (Türkçe'ye çevirince anahtar olduğunu tahmin ediyorum) ayarı ile belirttiğimiz tiplere uymayan argümanları da ayrıştırır. Örneğin `help` için boolean yerine string dersek ve `./fib --help -n 5 -c red --invalid 1234 -s 4321` şeklinde çalıştırırsak aşağıdaki gibi bir çıktı alacağız.

```zsh
./fib --help -n 5 -c red --invalid 1234 -s 4321
[help: true, order: 5, color: "red", invalid: "1234"]
[]
[]
```

`switches` yerine kullanabileceğimiz bir de `strict` tanımı var. Aralarındaki farkı değiştirerek görelim.

```zsh
./fib --help -n 5 -c red --invalid 1234 -s 4321
[help: true, order: 5, color: "red"]
["1234"]
[{"--invalid", nil}, {"-s", nil}]
```

`switches` `strict` in aksine tanımsız olanlar dahil tüm argümanları ayrıştırır, `strict` ise adını ve tipini belirlediklerimizi ayrıştırırak hatalı olanları da bildirir.

Kullanabileceğimiz tipler [dökümanda belirtildiği gibi](https://hexdocs.pm/elixir/OptionParser.html#parse/2-types) şunlardır:

```plain
:boolean - true/false değerler için, ve negatifleri için de `--no-` belirteci tanımlar.
:count - bir argümanın kaç defa gönderilebileceğini belirler

:integer - tam sayı değer
:float - ondalıklı sayı değer
:string - alfanümerik değer
```

Bir argümanı birden fazla tanımlarsak en son tanımlanan öncekileri ezer, öncekileri de saklamak istiyorsak `keep` kullanabiliriz, örneğin `switches`'i aşağıdaki şekilde değiştirelim.

```ex
[switches: [help: [:boolean, :keep], order: :integer, color: :string]]
```

Ve `--help` argümanından 2 tane gönderelim, geçerli argümanlar keyword listinde 2 tane help göreceğiz.

## Doğrulama ve varsayılan değerler

Örnek uygulamamız verilen order sırasındaki fibonacci sayısını verilen color renginde ekrana yazdıracak, bazı durumlarda aldığımız argümanları kontrol etmemiz gerekir. Örneğin fibonacci sayılarını hesaplamayı nerede bırakacağımızı uygulamamıza belirtmezsek canımız sıkılabilir. Color değeri geçerli bir renk olmazsa uygulamamız patlayabilir. Bu durumda zorunlu argümanların varlığını ve geçerliliğini kontrol edip, gerekiyorsa varsayılan değerler ile değiştirilmesi gerekebilir.

Bunu `optimus` gibi hex paketleri ile daha kolay yapabiliriz ama ben `OptionParser` ile kendimiz nasıl yapabiliriz bunu göstereceğim.

Modülümüze `@args` adında bir [`attribute`](https://elixir-lang.org/getting-started/module-attributes.html) ekleyelim ve `OptionParser`in options'ını bir fonksiyona çıkartalım.

```ex
defmodule Fib.CLI do
  @args [
    [order: [type: :integer, short: :n, required: true, default: 1, max: 1000]],
    [color: [type: :string, short: :c, default: "white"]]
  ]

  def main(args \\ []) do
    {opts, _, _} =
      args
      |> OptionParser.parse(options())

    opts
    |> validate
    |> IO.inspect()
  end

  defp options do
    @args
    |> Enum.reduce([strict: [], aliases: []], fn arg, acc ->
      name = hd(Keyword.keys(arg))
      options = arg[name]

      [
        strict: Keyword.merge(acc[:strict], [{name, options[:type]}]),
        aliases: Keyword.merge(acc[:aliases], [{options[:short], name}])
      ]
    end)
  end
  ...
```

`options/0` fonksiyonu aşağıdaki gibi bir keyword list döndürecektir.

```ex
[strict: [order: :integer, color: :string], aliases: [n: :order, c: :color]]
```

`validate/1` fonksiyonunu da aşağıdaki gibi private olarak yazalım.

```ex
defmodule Fib.CLI do
  ...
  defp validate(opts) do
    @args
    |> Enum.map(fn arg ->
      name = hd(Keyword.keys(arg))
      given = opts[name]
      options = arg[name]

      # required olup default değeri belirtilmemiş bir argüman ise RuntimeError fırlat
      given =
        case Keyword.get(options, :required, false) && Keyword.get(options, :default, false) &&
               is_nil(given) do
          true ->
            raise "#{String.capitalize(Atom.to_string(name))} attribute is required!"

          _ ->
            given
        end

      # default değeri belirtilmiş ve dışarıdan verilmemiş bir argüman ise defaultu yerleştir
      given =
        case Keyword.get(options, :default, false) && is_nil(given) do
          true ->
            options[:default]

          _ ->
            given
        end

      # type integer ve max değeri belirtilmiş ise
      case options[:type] == :integer && !is_nil(Keyword.get(options, :max, nil)) do
        true ->
          if given > options[:max] do
            raise "#{String.capitalize(Atom.to_string(name))} attribute could not be greater than #{
                    options[:max]
                  }"
          end

          given

        _ ->
          given
      end
    end)
  end
  ...
```

Şimdi `mix escript.build && ./fib` diyerek çalıştırdığımızda `(RuntimeError) Order attribute is required!` hatası verecek. `./fib -n 3` dersek `[order: 3, color: "white"]` diyecek.

## Fibonacci sayısını bulalım

Sayıyı bulup yazdırmak için aşağıdakileri fonksiyonları ekleyip `main/1` fonksiyonunda argümanları ayrıştırıp doğruladıktan sonra `run/1` fonksiyonunu çağıralım.

```ex
  ...
  defp run(args) do
    if Keyword.has_key?(IO.ANSI.__info__(:functions), String.to_atom(args[:color])) do
      color = apply(IO.ANSI, String.to_atom(args[:color]), [])
      IO.puts(color <> Integer.to_string(fib(1, 1, args[:order])))
    else
      raise "Color #{args[:color]} is invalid ANSI color!"
    end
  end

  defp fib(a, _, 0), do: a
  defp fib(a, b, n), do: fib(b, a + b, n - 1)
```

Ve deneyelim.

```zsh
mix escript.build && ./fib -n 1000 -c yellow

70330367711422815821835254877183549770181269836358732742604905087154537118196933579742249494562611733487750449241765991088186363265450223647106012053374121273867339111198139373125598767690091902245245323403501

./fib -n 5 -c red
8

./fib -n 5 -c asdf
** (RuntimeError) Color asdf is invalid ANSI color!
```

Bu örnek uygulamanın çalışan halini [github reposunda](https://github.com/murat/ex-fib-cli) bulabilirsiniz.

Faydalı bir içerik olduğunu umarak bir sonraki yazıda görüşmek üzere...
