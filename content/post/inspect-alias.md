---
title: Alias ne iş yapıyor?
date: 2017-09-02T10:57:12+30:00
authors:
  - Murat Bastas
---

Eğer benim gibi siz de bir başkalarının dotfiles repolarını kullanıyor iseniz bu konu size bazı durumlarda lazım olabilir.

Örneğin ben [Yan Pritzker](https://github.com/skwp)'e ait olan [YADR](https://github.com/skwp/dotfiles)(Yet another dotfiles repo)
deposunu kullanıyorum. Neden bunu kullanıyorsun diyen olursa cevabı Prezto. Her neyse, bildiğiniz gibi bu tür shell konfigurasyonlarında tonla ayar, eklenti, hack ve hayatımızı kolaylaştıran alias'lar var.

Bu alias'ları biz tanımlamadığımız için bazen benim gibi şuna benzer ikilemler yaşayabilirsiniz.

```zsh
$ gst
```

Git status a bakacaktım ama 😏

Tüm dosyaları git stash e aktardı. Emin olamadığım durumlarda hangi alias nereye gidiyor diye bakmak için çok basit ama bir şey var.

```zsh
$ type gst
gst is an alias for git stash

$ type gs
gs is an alias for git status
```

İşte bu kadar. Kafanız karıştığı zaman kullanacağınız alias'ın nerelere gideceğini type ile sorgulayabilirsiniz.
