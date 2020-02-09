---
title: "Windows Gelistirme Ortami"
date: 2020-02-09T14:34:05+03:00
authors:
  - Murat Bastas
---

Geliştirme için uzun süredir linux/unix kullanan birisi olarak desktop pc topladıktan sonra oyun oynama ihtiyacımı da görebileceğim bir ortam kurmaya çalıştım.
Steam'de linux uyumlu oyunlar olduğundan bana yeter diyerek manjaro linux ile geliştirme ortamımı kurdum, non-free nvidia driverlarını da kurduğum halde oyun oynarken donmalar, kasmalar yaşıyordum. Tüm çabalarıma rağmen ekran kartımın performansını linux üzerinde tam manasıyla değerlendiremedim. Haliyle windows ile dual-boot yapmak durumunda kaldım. Geliştirme zamanımı linuxta, oyun zamanımı windowsta değerlendireyim diyordum. Ama ben oyun oynamayalı oyunlar 20-100 GB arasında boyutlara çıkmış. Çok uzun indirme zamanları harcıyordum. Bu indirmeler sürerken işlerimi yapamaz durumda olduğum için acaba WSL ile windows üzerinde çalışamaz mıyım diye düşündüm. Ve istemeye istemeye windowsta geliştirme ortamı kurmaya çalıştım. Bu yazıda nasıl kurdum, ne zorluklarla karşılaştım, ve nasıl atlattığımdan bahsetmek istedim.

### WSL(windows subsystem for linux) kurulumu

WSL kurmak için windows denetim masasından, programlarda sol panelde windows özellikleri aç kapat menüsünden Windows Subsytem for Linux kutucuğunu işaretleyerek uygulamak yeterli. Biraz zaman alan bir işlemden sonra WSL kurulmuş oluyor, sonrasında windows mağazasından Ubuntu, Debian, Centos, Suse, Fedora, Kali vb dağıtımlardan birisini ve ya birkaçını seçip kurmak gerekiyor. Ben seçimimi Canonical ve Microsoft ortaklığından dolayı Ubuntu'dan yana yaptım. Diğerlerine nazaran daha iyi maintain ediliyor olabileceğini düşündüm. Kurulum yaklaşık 10 dakika kadar sürüyor.

![features1.png](/images/features1.png)

Kurulduktan sonra başlat menüsünde aşağıdaki gibi ubuntu kurulumunu gördüm, tıkladığımda sadece cmd'nin ubuntu/bash çalıştıran bir penceresi açıldı. Haliyle cmd bir gnome terminal ve ya iTerm gibi değil.

![start.png](/images/start.png)

Windowsta kullanabileceğim terminal uygulamalarını inceledim ve [Hyper](https://hyper.is) konfigüre etmeye çalıştım, fakat hyper'i zaten (başıma bir şey gelmeyecekse) sevmiyordum. [Alacritty](https://github.com/alacritty/alacritty) denedim, konfigürasyonuna uzun bir zaman harcadım, deneyen varsa bilir alacritty'de terminali screenlere ve ya panelere bölme olayı yok. zsh, tmux, ve vim konfigürasyonlarını yaptım bu pane eksikliğini tmux ile halletmeye çalıştım. Fakat mouse modu sebebini bilememekle birlikte çalıştıramadım. Paneler we windowlar arasında gezinti için klavye kısayollarını kullanmak zorunda kaldım. Ama en azından tab desteği olmasını dilerdim kullandığım terminalde. Mutlu olamadım bu yüzden alacritty'de. Alternatif araştırırken daha önce de karşıma çıkan windows terminal uygulamasını gördüm. Ben bu terminalin cmd ya da powershell olduğunu sanıyordum o yüzden denememiştim. Denediğimde en azından tab desteği olan windowu istediğim gibi boyutlandırıp kişiselleştirebildiğim bir terminalim oldu.

Subsystem içinde postgresql, redis, elasticsearch, elixir, go ve ruby kurulumlarını [asdf versiyon yöneticisi](https://murat.github.io/2019/04/13/hepsine-h%C3%BCkmedecek-tek-versiyon-y%C3%B6neticisi-asdf/) ile tamamladım. Docker kurdum fakat servisini bir türlü başlatamadım. Windows kullanan arkadaşlara sorduğumda linux kerneldeki sanallaştırma olaylarının windowsta olmamasından ötürü çalışmadığını öğrendim. Önce vazgeçmiştim. Bir süre sonra tekrar docker ihtiyacım olduğunda bir şekilde çalıştırabileceğine inanarak araştırmaya başladım. Ve bir yol buldum.

### WSL2 ye geçiş

Evet bir de WSL2 varmış. Kurulumu için [şurayı](https://docs.microsoft.com/en-us/windows/wsl/wsl2-install) okudum. Burada şöyle bir not var.

> WSL 2 is only available in Windows 10 builds 18917 or higher

Kendi buildime baktığımda daha düşük bir versiyon kurulu olduğunu gördüm, update, upgrade ne varsa denedim bu builde geçemedim. Sanırım bölgesel olarak dağıtılan bir update. Insider programına üye olursam alacakmışım bu yükseltmeyi. Geçtim. Bir kaç update / upgrade silsilesi daha yaşadıktan sonra:

```plain
Microsoft Windows [Version 10.0.19041.21]
(c) 2019 Microsoft Corporation. All rights reserved.

C:\Users\murat>ver

Microsoft Windows [Version 10.0.19041.21]
```

Ve:

```plain
C:\Users\murat>wsl -l -v
  NAME                   STATE           VERSION
* Ubuntu-18.04           Running         2
```

Şimdi dockerın çalışabileceğine inanıyordum, fakat daha yolun sonuna gelmemişim. Subsystem içinde docker kurmak yetmiyormuş, bir de docker for windows desktop client'ını kuracakmışım. Kurdum. Bu sefer servisi başlatabilip container çalıştıramamaya başladım. WSL de container çalıştırabilmek için docker desktop'ı edge channeldan kurmak gerekiyormuş. Yeniden kurdum ve ayarlarından resources menüsünde wsl integration görmeyi bekledim. Görmedim. Görebilmem için işlemcimde sanallaştırmanın aktif olması gerekiyormuş. Görev yöneticisinde performans sekmesinde CPU ya bakarken aşağıdaki gibi "Virtualization: Enabled" olması gerekiyormuş.

![cpu.png](/images/cpu.png)

Bunun için yine denetim masasında programlar menüsünden windows özelliklerine girip hyper-v yi aktif ettim, yeniden başlattım. Tekrar kontrol ettim, yok yine olmadı.

Enabled yazmıyordu Virtualization için. Araştırdım ve buldum, BIOS'ta işlemci ayarlarından sanallaştırma hizmetini açmak gerekiyormuş. Yolun başından itibaren çok kez vazgeçip linux'a dönmek istesem de azimle devam ettiğim için sonunda bu adımdan sonra docker edge client ayarlarında wsl integration'a baktığımda ubuntu'mu görebildim. Ve aşağıdaki gibi aktif ettikten sonra subsystem içinde de docker'i kullanabilmeye başladım.

### Sonuç

VSCode'u açtığımda zaten WSL'i kendisi tanıyıp, WSL içinde çalışabilmek için gerekli bir kaç extensionu kurdu, ben de ihtiyacım olan diğer extensionları kurdum. Her birinde ekstra olarak WSL üzerinde kurma seçeneği çıktı. Kurulumları yaptıktan sonra günlük hayatımda linuxta yaptığım gibi geliştirme yapabilmeya başladım.

Kan ve gözyaşı dolu bu sürecin sonunda yaklaşık 1 aydır sadece windows üzerinde geliştirme yapıyorum. Ve mutsuz değilim.

Not: aptal aptal şeyler dediysem windows hakkında, cahilliğimdendir. 🤷‍♂️ Esen kalın.