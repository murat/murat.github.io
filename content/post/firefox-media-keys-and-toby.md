---
title: "Firefox medya tuşları ve toby eklentisi"
date: 2020-04-08T11:18:05+03:00
authors:
  - Murat Bastas
---

Klavyelerimizin fonksiyon tuşları, medya tuşları genellikle spotify, itunes, vlc player gibi müzik, video uygulamalarında süper çalışıyor. Ne zamandır olduğunu bilmesem de bir süredir youtube videolarını kontrol etmeye de yarıyordu. Firefox kullanıyordum uzun bir süredir ve youtube music kullanmaya başladıktan sonra media tuşlarının youtube music'e müdahale edemediğini farkettiğimde sıkıcı bir süreç ile google chrome'a geçiş yapmak zorunda kalmıştım. Bugün firefox'ta chrome'un `chrome://flags` ine benzer bir konfigurasyon arayüzü olan `about:config` i kurcalarken aşağıdaki ayarı buldum.

```
media.hardwaremediakeys.enabled = true
```

Ve artık memnuniyetle firefox kullanmaya dönebilirdim. Derken uzun bir süredir chrome'da bookmark yöneticisi olarak kullandığım toby eklentisi ile sorun yaşadım, eskiden bunun yerine oneTab kullanıyordum. Ama toby tüm cihazlarımla sync olduğu için daha kullanışlıydı ve chrome'da yeni alışkanlığım haline gelmişti. Toby malesef firefox eklenti mağazasında yok, ama sitesinde de gettoby.com/firefox şeklinde de bir bağlantı var, fakat çalışmıyor. Archive.org da o sayfanın snapshotlarını mı aramadım, twitter'dan toby'e mention mu atmadım, fakat çözüm bulamadım ve oneTab kullanmaya döndüm. Şimdi sıradaki sorunum toby'den export ve oneTab'e import. Bunun için de toby'nin localStorage'de nerede data sakladığını araştırdım, derken armut pişip ağzıma düştü ve [şu gist](https://gist.github.com/krishpop/8a954b171a5403117bf0f2fdda0a8e90) ile karşılaştım. Json dosyasına aldıktan sonra oneTab'e import etmek kaldı. oneTab aşağıdaki şekilde bir import istiyor.

```
https://murat.github.io | Murat Bastas's homepage
https://github.com/murat | murat (Murat Bastas)

https://twitter.com/muratbastas | Murat Bastas (@muratbastas)
```

Boş satırlar farklı bir grup oluyor. Fakat toby'den aldığım export bir json dosyası ve aşağıdakine benziyor.

```json
{
  "lists": [
    {
      "title": "Mar 10 at 12:27",
      "cards": [
        {
          "title": "Murat Bastas's homepage",
          "url": "https://murat.github.io",
          ...
        },
        ...
    },
    ...
  ]
}
```

Bu json'u da jq aracı ile oneTab formatına dönüştürmek için aşağıdaki gibi parse ettim,

```shell
cat backup.json | jq --raw-output '.lists[].cards[] as $card | "\($card.url) | \($card.title)"'
```

Çıktısını da oneTab'den import ettikten sonra başarıyla bookmark listemi de almış oldum. Sonuç olarak şuan firefox kullanmam önünde hiç bir engel bırakmamış bulunuyorum. Yaşasın Quantum 😍.
