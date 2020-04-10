---
title: "WSL2 içinde linux desktop ortamı, GTK uygulamaları ve JetBrains IDE'leri"
date: 2020-04-09T21:01:05+03:00
authors:
  - Murat Bastas
---

Tam 2 ay önce windows içinde linux geliştirme ortamımı nasıl kurduğumu anlattığım [bir yazı yazmıştım.](https://murat.github.io/2020/02/09/windows-gelistirme-ortami/) Bu yazıda windows 10 insider ile WSL2 kurulumu, intel sanallaştırma hizmetlerinin BIOS üzerinden aktif edilmesi, docker for windows edge kurulumu(artık edge kanalından kurmak zorunda değilsiniz) ve wsl içinde docker kullanımını anlatmıştım. Bu yazımda da WSL içinde bir linux desktop ortamı kurup, gui uygulamalarını nasıl windows içinde çalıştırabileceğimizi anlatacağım.

Öncelikle linux görüntü sunucusu olan "X server" nedir diye merak edenlere bir [tr.wiki linki](https://tr.wikipedia.org/wiki/X_Pencere_Sistemi) bırakayım. Windows için ise bir kaç X Server bulunuyor. Bunlardan bazıları:

1. [vcxsrv](https://sourceforge.net/projects/vcxsrv/)
2. [xming](https://sourceforge.net/projects/xming/)
3. [x410](https://x410.dev/)

Benim tercihim birincisi. sourceforge.net üzerinden programı indirip kurduktan sonra WSL içinden de kullanacağımız DISPLAY serverin adresini belirtmemiz gerekiyor. Bunun için zsh kullanıyorsanız `~/.zshrc` ve ya bash için `~/.bash_profile` a aşağıdaki satırları eklemeliyiz.

```
export DISPLAY=$(cat /etc/resolv.conf | grep nameserver | awk '{print $2; exit;}'):0.0
export LIBGL_ALWAYS_INDIRECT=1
```

XLaunch ile görüntü sunucumuzu başlattıktan aşağıdaki adımlarla başlattıktan sonra GUI uygulamalarımızı WSL den başlatabiliriz.

![x1](/images/xsrvr1.jpg)
![x2](/images/xsrvr2.jpg)

Linux masaüstü ortamları hakkında detaylı bilgiler almak için araştırabilirsiniz, ben hızlıca en hafif masaüstü ortamlarından birisi olan xfce kuruyorum.

```
sudo apt update && sudo apt upgrade -y && sudo apt install xfce4
```

Ve sonra masaüstü ortamını aşağıdaki komutla başlatabilir, bir windows penceresi içinde kullanabilirsiniz.

```
startxfce4
```

![xfce4](/images/xfce4.jpg)

Peki neden bunu yapıyoruz. Çünkü (neden yapmayalım :) linux ortamında uygulamalar geliştirmek için hali hazırda kullanabildiğimiz tek editor VSCode. Çok başarılı bir şekilde bir demet WSL eklentisi ile linux içinde uygulamalar geliştirebiliyoruz. Ama kullanabileceğimiz bir IDE şuan için malesef yok. Bunun için: 

## Snapd ve JetBrains IDE kurulumu

Bu konuda anlatacak pek bir şey yok, snapd kurmak aşağıdaki komutu çalıştırmak kadar basit.

```
sudo apt install snapd

sudo service snapd start
```

Snapd ile JetBrains IDE'lerini hiç zorlanmadan kurabilir ve sadece VSCode kullanma zorunluluğundan kurtulabilirsiniz.

```
sudo snap install golang --classic
```

Birilerinin hayatını kolaylaştırması dileğiyle..
