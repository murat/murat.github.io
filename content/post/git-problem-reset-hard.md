---
title: git reset --hard ile commitlenmemiş dosyaları kaybedersek?
date: 2017-04-02T20:30:33+30:00
authors:
  - Murat Bastas
---

Çok acı bir anı sahibi olabilirsiniz. Evet bu acı anımı anlatacağım şimdi...

Yine bir proje üzerindeyim(deneysel). Uğraştım didindim 2 gece boyunca ve başarılı sonuç aldım. Github'da bir repo açayım(private) buna dedim ve açtım. Remote url'mi ekledim ve...

```zsh
git add .
#........
```

Dosyaları ekledikten sonra dur geri alayım önce vendoru vs ignore edeyim dedim.

```zsh
git reset --hard
#......... 😨
```

Dedim ve o an malum pişmanlığı tüm dosyaları kaybederek yaşadım. Soğuk soğuk terlemeye başladım. Hemen google _**`recover uncommitted files after git reset --hard`**_ falan filan...

[Şu stack sayfasını](http://stackoverflow.com/a/7376959/6668082) buldum.

Çok şükür kaybettiğim 3-5 dosya(az sayıda) olduğu için **dangling blob** neyse oradan teker teker `git show xxxxxxx` diyerek kurtarmak zorunda olduğum dosyaları buldum kurtardım.
