---
title: 'Nasıl: Her terminal için ayrı zsh prompt kullanılır?' 
date: 2020-06-16T12:27:12+30:00
authors:
  - Murat Bastas
---

Shell ortamı için zsh mı kullanıyorsunuz? Bir de powerlevel9(10)k mı seviyorsunuz? Ama vscode, idea ve terminal unicode karakterleri farklı mı render ediyor? O zaman size bir ipucu vereyim...

Ben gnome terminal, vscode integrated terminal ve idea terminalleri kullanıyorum. Fakat her biri zsh prompt'umda farklı şekilde render ediliyor.
Hem de font birebir aynı olmasına rağmen. Bu durumda editor/ide lerin içinde çirkin bir terminal görmemek için şöyle bir çözüm buldum. ~/.zshrc dosyanızda aşağıdaki if koşulunu kullanarak ortamlarda farklı zsh promptları kullanmanız mümkün.

```shell script
if [[ "$TERM_PROGRAM" == "vscode" ]]; then
    source ~/.vscode-icin-prompt.zsh
elif [[ $__INTELLIJ_COMMAND_HISTFILE__ ]]; then
    source ~/.idea-icin-prompt.zsh
else
    source ~/.diger-terminal-icin-prompt.zsh
fi
```
