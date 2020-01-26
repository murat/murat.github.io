---
title: "DO Ubuntu sunucu kurulumu (part 1)"
date: 2017-01-01T17:07:12+30:00
authors:
  - Murat Bastas
---

**Seri:**

- [DO Ubuntu sunucu kurulumu (part 1)](/2017/01/01/do-ubuntu-sunucu-kurulumu-part-1/)
- [DO Ubuntu sunucu kurulumu (part 2)](/2017/03/25/do-ubuntu-sunucu-kurulumu-part-2/)

Geçtiğimiz hafta x firmasından aldığımız sunucu hizmetini sonlandırıp kendi sunucumuzu digitalocean'da kurmaya karar verdik. Eski sunucumuz CentOS 6.7 x64 idi. Bu sefer ubuntu kurmaya karar verdim. Digitalocean'da 20 dolarlık ubuntu 14.04 makine oluşturduktan sonra `ssh root@xxx.xxx.xxx.xxx` ile sunucuya bağlandım. Sonra…

## Ssh bağlantı yapılandırmam

İlk iş root kullanıcısını kullanmamak için yeni bir kullanıcı oluşturup **sudoers‘e** eklemek oldu. **murat** diye bir kullanıcıyı aşağıdaki komutlarla oluşturdum.

```zsh
adduser murat
# burada soracağı parolayı unutmayalım

gpasswd -a murat sudo
sudo - murat
```

Uygulamalarımı bu kullanıcı hesabını kullarak deploy edeceğim için github, gitlab, bitbucket repolarıma erişebilmesi için ssh-keygen ile key oluşturdum.

```zsh
# ssh-keygen output
Generating public/private rsa key pair.
Enter file in which to save the key (/Users/murat/.ssh/id_rsa):
```

Enter, enter diye diye geçtim. `cat ~/.ssh/id_rsa.pub` ile key çıktısını alıp github, gitlab ve bitbucket hesaplarıma ekledim.

Bu kullanıcı ile bu sunucuya parola girmeden giriş yapmak için yerel bilgisayarımda aşağıdaki komutu çalıştırdım.

```zsh
ssh-copy-id murat@xxx.xxx.xxx.xxx
```

**murat** kullanıcısı için tanımladığım parolayı sordu, doldurdum devam ettim ve yerel ssh-key imi murat kullanıcısının ev dizininde .ssh/authorized_keys dosyası içine ekledi. yerel bilgisayarımdan `ssh murat@xxx.xxx.xxx.xxx` yazarak parola girmeden sunucuda oturum açtım. Tabi yerel bilgisayarımda `~/.ssh/known_hosts` dosyama eklemek isteyip istemediğimi sordu isterim diyerek onayladım.

Sonra sunucumda `sudo vim /etc/ssh/sshd_config` komutu ile ssh ayarlarını değiştirmeye başladım. Fazla da detaya gerek yok iki kaydı güncelledim.

```conf
#...
Port 2222 # ve ya dört haneli ne dilerseniz
PermitRootLogin no # varsayılanı yes idi
#...
```

Varsayılan portu değiştirip root olarak login yapmayı engelledim. Değişikliklerin aktif edilmesi için `service ssh restart` komutunu çalıştırdım.

Sonra yerel bilgisayarımda `~/.ssh/config` dosyama aşağıdaki gibi yapılandırma ekledim.

```conf
Host do
  HostName xxx.xxx.xxx.xxx
  Port 2222
  User murat
```

Bu ayar sayesinde konsolumdan `ssh do` yazıp enter yapmam sunucuda oturum açmama yetti.

## Bölge ve tarih

```zsh
sudo dpkg-reconfigure tzdata
```

Komutunu çalıştırıp ilk listeden Europe ikinci listeden Istanbul seçtim.

## Swap dosyası

**Swap dosyası nedir?**

_Linux işletim sistemlerinde sahip olduğunuz fiziksel RAM ‘în daha fazla belleğe ihtiyaç duyduğunda hard diskinizi kullanarak size fiziksel bellek gibi kullanmanızı sağlayan sisteme Swap deriz._

Swap dosyası oluşturmak için alttaki komutları çalıştırdım.

```zsh
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
sudo sh -c 'echo "/swapfile none swap sw 0 0" >> /etc/fstab'
```

## Güvenlik duvarı

Ubuntu da **ufw** varsayılan güvenlik duvarı olarak kurulu oluyormuş. Eğer kurulu olmadığı istisnai bir durum varsa `sudo apt-get install ufw` ile kuruyormuşuz.

Eğer dropletimizi IPV6 ile kurduysak ufw konfigurasyonunda IPV6 yı aktif ediyoruz. `sudo vim /etc/default/ufw` yazarak aşağıdaki değişikliği yapıyoruz.

```zsh
#...
IPV6=yes
#...
```

Normal şartlarda ssh bağlantılarına izin vermek için `sudo ufw allow ssh` komutu çalıştırılıyor fakat bu varsayılan port 22 den bağlanmaya izin veriyor. Eğer varsayılan portu değiştirmediyseniz böyle izin verebilirsiniz. Eğer değiştirdiyseniz,

```zsh
sudo ufw allow PORTNO ## Port ne yaptıysanız o
```

Ek olarak

```zsh
sudo ufw allow http
sudo ufw allow https
sudo ufw allow smpt
sudo ufw allow ftp
sudo ufw allow postgresql
sudo ufw allow 21/tcp
sudo ufw allow 143/tcp
sudo ufw allow 587/tcp
sudo ufw allow 443/tcp
sudo ufw allow 5432/tcp
```

Ekleyip

```zsh
sudo ufw enable
```

Komutu ile güvenlik duvarını aktif ettim. Değişiklikleri kontrol etmek için

```zsh
sudo ufw status verbose
```

Komutunu çalıştırdım, örnek olarak aşağıdaki gibi bir çıktı aldım.

```zsh
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22                         ALLOW IN    Anywhere
80/tcp                     ALLOW IN    Anywhere
443/tcp                    ALLOW IN    Anywhere
587/tcp                    ALLOW IN    Anywhere
22 (v6)                    ALLOW IN    Anywhere (v6)
80/tcp (v6)                ALLOW IN    Anywhere (v6)
443/tcp (v6)               ALLOW IN    Anywhere (v6)
587/tcp (v6)               ALLOW IN    Anywhere (v6)
```

**Güvenlik duvarını devre dışı bırakmak için**

```zsh
sudo ufw disable
```

**Güvenlik duvarı ayarlarını sıfırlamak için**

```zsh
sudo ufw reset
```

komutlarını çalıştırabilirsiniz.

**Güvenlik duvarı kurallarını silmek için**

```zsh
sudo ufw status numbered # diyerek rule numaralarını görür

sudo ufw delete RULE_NUMBER # diyerek de belli bir kuralı silersiniz
```

Daha detaylı bilgi için [buraya](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-ubuntu-14-04) bakabilirsiniz.

## VestaCP kurulumu

Vesta ücretsiz / **opensource** hosting paneller arasında benim gözümde en iyi üç'ten en makul olanı. Diğer seçenekler [ajenti v panel](http://ajenti.org/) ve [centos web panel](http://centos-webpanel.com/). Centos web panel centos ve redhat haricindeki dağıtımlarda çalışmadığı için direkt elendi gözümde zaten. Ajenti ile ise iki ayrı sunucu kurup kullandım fakat deneyimsiz birisine sunulamayacağına karar verdim. Sonuçta bu paneli sadece ben kullanmayacağım. Bu yüzden VestaCP de karar kıldım.

Vesta installer dosyasını wget ile çektim. Sonra da [şuradan](http://vestacp.com/install/) advanced install komutu türettim ve başlattım kurulumu. Yaklaşık 10 dk kadar sürdü.

```zsh
bash vst-install.sh --nginx yes --apache yes --phpfpm no --named yes --remi yes --vsftpd yes --proftpd no --iptables yes --fail2ban yes --quota no --exim yes --dovecot yes --spamassassin yes --clamav yes --mysql yes --postgresql yes --hostname domain.com --email murat@domain.com
```

İşlem bittiğinde bana admin kullanıcı bilgilerimi verdi ve giriş yaptım. Artık elimde şöyle bir sunucu vardı.

- Apache2 + nginx reverse proxy
- Php 5.5.x
- Exim4
- ClamAV + SpamAssassin
- Firewall
- Hosting Panel

Şimdi geriye droplet'in hostname ayarını yapmak ve domainleri yönlendirmek kaldı.

Sonra kullanıcı, website, e-mail, veritabanı ekleme işlemlerini kolaylıkla halledebilirim. Bu kısım sadece VestaCP kullanmaktan ibaret. Buraya daha sonralardan girerim.

Şimdi yazı çok uzadığı için hostname, dns ve e-mail ayarlarımı yeni bir yazı ile anlatacağım. Esen kalın…
