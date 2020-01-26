---
title: "Elixir geliştirme ortamım"
date: 2019-05-19T14:00:25+30:00
authors:
  - Murat Bastas
---

Merhabalar, bu yazıda Elixir geliştirirme ortamımı nasıl kurduğumdan, hangi araçları kullandığımdan vb konulardan bahsedeceğim. [Elixir forum](https://elixirforum.com/) da hangi editor, hangi IDe, hangi versiyon yöneticisi vb çeşitli sorular ve cevaplar var. Ben hangilerini seçtim?

## Versiyon yönetimi

Elixir, Erlang versiyon yönetici olarak [asdf-vm](https://asdf-vm.com/) kullanıyorum. Bundan [şu yazıda da](/2019/04/13/hepsine-h%C3%BCkmedecek-tek-versiyon-y%C3%B6neticisi-asdf/) bahsetmiştim.

asdf kurduysanız Elixir ve Erlang kurmak için aşağıdaki komutları çalıştırmanız yeterli gelecektir.

```zsh
asdf plugin-add erlang && asdf plugin-add elixir
asdf install erlang 21.3
asdf install elixir 1.8.1
asdf reshim

elixir --version
Erlang/OTP 21 [erts-10.3] [source] [64-bit] [smp:4:4] [ds:4:4:10] [async-threads:1] [hipe]

Elixir 1.8.1 (compiled with Erlang/OTP 20)
```

## Editor & IDE

Editor olarak çoğunlukla Visual Studio Code kullanıyorum, fakat terminal üzerinde çalışıyorsam vim ile işlerimi kolaylıkla hallediyorum. Eğer bir IDE'ye ihtiyacım olursa ki pek olmuyor RubyMine'a [elixir plugini](https://plugins.jetbrains.com/plugin/7522-elixir) kurarak kullanıyorum.

### Visual Studio Code

Elixir desteği katmak için [ElixirLS](https://github.com/JakeBecker/elixir-ls) ve [EEx snippets](https://github.com/stefanjarina/vscode-eex-snippets) kullanıyorum. Ayarlarım da aşağıdaki şekilde.

```json
{
  "editor.fontFamily": "FuraCode Nerd Font",
  "editor.fontLigatures": true,
  "editor.lineHeight": 24,
  "editor.formatOnSave": true,
  "editor.formatOnPaste": true,
  "files.trimTrailingWhitespace": true,
  "files.insertFinalNewline": true,
  "files.associations": {
    "HTML (Eex)": "html",
    "*.leex": "html"
  },
  "emmet.includeLanguages": {
    "HTML (Eex)": "html"
  }
}
```

### Vim

Elixir desteği katmak için aşağıdaki pluginleri kullanıyorum.

```vimrc
Plugin 'majutsushi/tagbar'
Plugin 'mmorearty/elixir-ctags'
Plugin 'elixir-editors/vim-elixir'
Plugin 'mhinz/vim-mix-format'
```

Ayarlarım da aşağıdaki şekilde.

```vimrc
# .vimrc

...
set formatprg=mix\ format\ -
let g:mix_format_on_save = 1
let g:mix_format_silent_errors = 1
let g:mix_format_options = '--check-equivalent'

map <C-t> :TagbarToggle<CR>

let g:tagbar_ctags_bin = '/usr/local/bin/ctags'

let g:tagbar_type_elixir = {
    \ 'ctagstype' : 'elixir',
    \ 'kinds' : [
        \ 'f:functions',
        \ 'functions:functions',
        \ 'c:callbacks',
        \ 'd:delegates',
        \ 'e:exceptions',
        \ 'i:implementations',
        \ 'a:macros',
        \ 'o:operators',
        \ 'm:modules',
        \ 'p:protocols',
        \ 'r:records',
        \ 't:tests'
    \ ]
  \ }
```

NERDTree ile tadından yenmiyor. :)

## Debug

Debug etmek için [Elixir dökümantasyonunda](https://elixir-lang.org/getting-started/debugging.html) anlatılan yollar haricinde bir yola başvurmam gerekmedi şimdiye kadar, `IEx` her zaman yeterli geldi.

Bunlar dışında phoenix ile uygulama geliştirirken javascript paket yöneticisi olarak `yarn` kullanmayı tercih ediyorum.

Hangi araçları kullanacağını şaşıran olursa faydası olması dileğiyle...
