---
title: "Hepsine hükmedecek tek versiyon yöneticisi 'ASDF'"
date: 2019-04-13T11:13:24+30:00

---

Bu devirde yazılım geliştirmek ile meşgul herkes sanırım yazdığı dil için versiyon yöneticisi kullanıyordur. Çünkü uygulamalarımızın hepsi aynı versiyon ile çalışmak zorunda değil.
Örneğin, ruby ile programlama yapıyorsanız rbenv ve ya rvm, nodejs yazıyorsanız nvm kullanıyorsunuzdur. Kullanmıyor da olabilirsiniz tabi, sizin tercihiniz. Fakat eğer kullanıyorsanız kendinize bir geliştirme ortamı kurmak çok zahmetli bir hal alıyor olabilir. Bu yüzden bizi bu dertten kurtaracak, yakın zamanda keşfettiğim açık kaynak kodlu `asdf` isimli versiyon yöneticisini tanıtmak istiyorum size.

## Nedir `asdf`?

`asdf` şurada listelendiği gibi en az bir versiyon yöneticisine sahip olan neredeyse tüm geliştirme araçlarının versiyon yönetimini kurduğunuz pluginler ile yönetmeye yarayan bir geliştirici aracıdır. Örneğin benim gibi aynı zamanda ruby, php, elixir ve javascript dilleri ile geliştirme yapıyor ve projelerinizin hepsi her zaman aynı versiyon dili kullanmıyor ise bilgisayarınızda rbenv, nvm, phpbrew vs bir sürü araç kurulmuş durumdadır. `asdf` bana bütün bu araçları bir kenara atıp sadece tek bir komut satırı uygulaması ile bütün kullandığım programlama dillerinin versiyon yönetimini yapabilme imkanı veriyor. İşte hepsine hükmedecek tek versiyon yöneticisi budur. :)

## Nasıl kurulur?

Aslında çok basit, benim tekrar yapmama hiç gerek yok, [şuradan](https://asdf-vm.com/#/core-manage-asdf-vm?id=install-asdf-vm) ilerleyebilirsiniz.

## Nasıl kullanılır?

Ben kendi bilgisyarımda nasıl kullandığımı anlatayım öncelikle. Biraz can sıkıcı olsa da geliştirme ortamını baştan yapılandıracağız. Ruby için rbenv kullanıyordum, öncelikle tüm rbenv versiyonlarını ve gemleri kaldırdım. Bunun için aşağıdaki şekilde bir bash script kullandım.

**Dikkat!** çalıştırıldığında sistemdeki local gemler hariç tüm ruby versiyonlarındaki tüm gemleri kaldıracaktır.

```zsh
uninstall_gems() {
  for gem in `gem list --no-versions`; do
    gem uninstall $gem -aIx
  done
}

RUBIES=`ls $(rbenv root)/versions`
for ruby in $RUBIES; do
  rbenv local $ruby
  uninstall_gems
done

rm -rf `rbenv root`
```

`~/.zshrc` ve ya `~/.bashrc` dosyanızdan da rbenv init çağrısını kaldırdıktan sonra `source ~/.zshrc` ve ya `source ~/.bashrc` çalıştırıp son olarak da rbenv ile ilişkimizi aşağıdaki şekilde kesebiliriz.

```zsh
brew uninstall rbenv
```

Diğer versiyon yöneticilerini ve versiyonları nasıl kaldırdığımı tek tek anlatmayacağım.

### Plugin kurulumu

Çok basit bir şekilde asdf e plugin repository si eklemek mümkün, örneğin elixir, erlang ve ruby için:

```zsh
asdf plugin-add erlang && asdf plugin-add elixir && asdf plugin-add ruby
```

Eğer official bir asdf repository si değil ise eklediğimiz repository nin git adresini aşağıdaki şekilde ekliyoruz.

```zsh
asdf plugin-add <name> <git-url>
```

Çekilmiş repository leri de aşağıdaki komut ile listeleyebiliriz.

```zsh
asdf plugin-list --urls
```

### Versiyon kurulumu

Eklediğimiz repository lerden istediğimiz versiyonu aşağıdaki gibi basit komutlar ile kurabiliriz.

```zsh
asdf install ruby 2.5.0 && asdf install erlang 21.3 && asdf install elixir 1.8.1
```

Dahası için asdf'nin süper dökümantasyonuna başvurabilirsiniz.

Happy coding... 🍻
