---
title: "DO Ubuntu sunucu kurulumu (part 2)"
date: 2017-03-25T09:08:12+30:00
authors:
  - Murat Bastas
---

**Seri:**

- [DO Ubuntu sunucu kurulumu (part 1)](/2017/01/01/do-ubuntu-sunucu-kurulumu-part-1/)
- [DO Ubuntu sunucu kurulumu (part 2)](/2017/03/25/do-ubuntu-sunucu-kurulumu-part-2/)

Aradan bir hayli zaman geçti fakat yoğunluktan devam etme fırsatım olmadı. Önceki bölümde en son vestacp kurulumunu yapıp bırakmıştık. Bununla bitmedi tabii ki. Sırada hostname ayarlayıp domain yönlendirmek ve hostinglerini açarak e-mail, veritabanı vs açmak kaldı. Vesta üzerinden yapılacak işler(hosting, e-mail, veritabanı aç kapa) çok basit olduğu için daha önce belirttiğim gibi o konuya girmeyeceğim.

Bu yazıda sadece hostname, dns ve e-mail ayarlarını yapmayı anlatacağım.

Sunucuya tekrar bağlanıyoruz `/etc/hostname` dosyasına bakarsanız mevcut hostname in ne olduğunu görürsünüz. Bu dosyadaki kaydı değiştirerek hostname i değiştirmiş oluruz fakat ben digitalocean üzerinde droplet adını değiştirerek hostname i değiştirdim. Bu şekilde yaptığımda zaten `/etc/hostname` değişmiş oldu. Örneğin `host.domain.com`

Şimdi digitalocean networking panelinden domainimizi sunucumuza eklememiz lazım.

![https://s18.postimg.org/5ijsquwjd/Ekran_Resmi_2017-03-25_12.26.08.png](https://s18.postimg.org/5ijsquwjd/Ekran_Resmi_2017-03-25_12.26.08.png)

Domain ekledikten sonra **manage domain** menüsünden aşağıdaki A ve CNAME kayıtlarını ekleyeceğiz.

```plain
domain.com. 1800 IN A SERVER_IP
mail.domain.com. 1800 IN A SERVER_IP
ns1.domain.com. 1800 IN A SERVER_IP
ns2.domain.com. 1800 IN A SERVER_IP
host.domain.com. 1800 IN A SERVER_IP
www.domain.com. 1800 IN CNAME domain.com.
```

Domain'i nereden aldıysak oranın yönetim arayüzüne girerek nameserverlarımızı aşağıdaki gibi ayarlamalıyız.

- **ns1.digitalocean.com**
- **ns2.digitalocean.com**
- **ns3.digitalocean.com**

Domain nameserverları değiştiğinde öğrendiğim kadarıyla ICANN tarafından bildirilen süre maksimum 72 saat imiş. Ama genelde en fazla 1 saat içinde nameserverlar yönlenmiş olur. Kontrol için ```whois domain.com``` şeklinde sorgulayabilirsiniz. DNS önbelleğinizi de temizlemekte fayda var daha erken görebilmek için.

Domain yönlendiği zaman `http://host.domain.com:8083` adresine girdiğinizde bağlantı güvenli değil uyarısı alacaksınız ama ilerle diyerek vesta login sayfasını göreceksiniz. Kurulumda belirttiğiniz parola ile vestaya giriş yapın.

Vesta panel üzerinden kullanıcı ve website oluşturun. Website oluşturduktan sonra DNS menüsüne gelip websitenin yanındaki **LIST RECORDS** butonuna tıklayın. Vesta bizim için bazı keyler oluşturmuş olacak. Buradan DKIM, DMARC, SPF kayıtlarının değerlerini kopyalayın ve digitalocean networking panelinden aşağıdaki gibi ekleyin.

```plain
domain.com. 1800 IN TXT "v=spf1 a mx ip4:SERVER_IP ?all"

mail._domainkey.domain.com. 1800 IN TXT "v=DKIM1; k=rsa; p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQC4NeHh2+tlEumhfm0mv0MgJbtvTdLOSPOx/jQxinqv9Ek8Lvq3QN8rNJcyBG1tbwnyueu/mhzUGfIrtnNONgrM5irtz5QDWUgEzAuqk/a/RutyVhSNR135eVU6/tn/WIZ4407K5+etoPkUx8bmfs/X/TyUxuZIFadY/4AA513n5wIDAQAB"

_dmarc.domain.com. 1800 IN TXT "v=DMARC1; p=quarantine; rua=mailto:info@domain.com; ruf=mailto:info@domain.com; fo=0:1:d:s; aspf=s"
```

Ardından MX kayıtlarını aşağıdaki gibi ekleyin.

```plain
domain.com. 1800 IN MX 1 mail.domain.com.
domain.com. 1800 IN MX 5 mail.domain.com.
domain.com. 1800 IN MX 10 mail.domain.com.
```

Ve şimdi mxtoolbox ve ya intodns gibi servislerden domain kayıtlarınızı sorgulayabilirsiniz.

Eğer sıradışı bir kayıt veya uyarı görünürse yorum ile bildirirseniz yardımcı olmaya çalışacağım. Ben ek olarak [mail-tester](https://www.mail-tester.com/) servisinden mailler ile alakalı olan kayıtlarımı test ettim. Ve 10 üzerinden 9.8 e kadar puan aldım bu kayıtlarla.

Umarım bu dizi birilerine faydalı olur. Esen kalın. 🙏🏻
