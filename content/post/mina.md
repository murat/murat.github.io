---
title: "Başarılı bir capistrano alternatifi: mina"
date: 2016-10-16T19:20:00+03:00
authors:
  - Murat Bastas
---

Merhaba, çok uzun zaman önce planladığım ama bir fırsat bulup da yazmaya başlayamadığım ilk yazımı yazmanın vakti geldi de geçiyor.

Bu yazımda Ruby on Rails ile yazdığım projemi capistrano ile deploy etmeye çalışırken daha basit configure edebildiğim mina ile nasıl deploy ettiğimi anlatacağım.

## Hazırlık

Öncelikle **mina** sistemimizde kurulu olmalı. `gem install mina` komutu ile mina gem'ini kuruyoruz. Her hangi bir izin(permission) hatası alırsanız gem'i **sudo** ile kurabilirsiniz. Benim deploy edeceğim projenin adı **projem** olsun. Eğer sizin mevcut bir projeniz yoksa;

```zsh
rails new projem --skip-bundle && cd projem
echo "gem 'therubyracer', platforms: :ruby" >> Gemfile
bundle install
```

Projemiz hazır iken aşağıdaki gibi mina için config/deploy.rb dosyamızı oluşturuyoruz.

```zsh
# /local/makine/projeler/projem
mina init
```

deploy.rb dosyasının içeriği varsayılan olarak alttaki şekilde oluyor.

```zsh
require 'mina/bundler'
require 'mina/rails'
require 'mina/git'
# require 'mina/rbenv'  # for rbenv support. (http://rbenv.org)
# require 'mina/rvm'    # for rvm support. (http://rvm.io)

# Basic settings:
#   domain       - The hostname to SSH to.
#   deploy_to    - Path to deploy into.
#   repository   - Git repo to clone from. (needed by mina/git)
#   branch       - Branch name to deploy. (needed by mina/git)

set :domain, 'foobar.com'
set :deploy_to, '/var/www/foobar.com'
set :repository, 'git://...'
set :branch, 'master'

# For system-wide RVM install.
#   set :rvm_path, '/usr/local/rvm/bin/rvm'

# Manually create these paths in shared/ (eg: shared/config/database.yml) in your server.
# They will be linked in the 'deploy:link_shared_paths' step.
set :shared_paths, ['config/database.yml', 'config/secrets.yml', 'log']

# Optional settings:
#   set :user, 'foobar'    # Username in the server to SSH to.
#   set :port, '30000'     # SSH port number.
#   set :forward_agent, true     # SSH forward_agent.

# This task is the environment that is loaded for most commands, such as
# `mina deploy` or `mina rake`.
task :environment do
  # If you're using rbenv, use this to load the rbenv environment.
  # Be sure to commit your .ruby-version or .rbenv-version to your repository.
  # invoke :'rbenv:load'

  # For those using RVM, use this to load an RVM version@gemset.
  # invoke :'rvm:use[ruby-1.9.3-p125@default]'
end

# Put any custom mkdir's in here for when `mina setup` is ran.
# For Rails apps, we'll make some of the shared paths that are shared between
# all releases.
task :setup => :environment do
  queue! %[mkdir -p "#{deploy_to}/#{shared_path}/log"]
  queue! %[chmod g+rx,u+rwx "#{deploy_to}/#{shared_path}/log"]

  queue! %[mkdir -p "#{deploy_to}/#{shared_path}/config"]
  queue! %[chmod g+rx,u+rwx "#{deploy_to}/#{shared_path}/config"]

  queue! %[touch "#{deploy_to}/#{shared_path}/config/database.yml"]
  queue! %[touch "#{deploy_to}/#{shared_path}/config/secrets.yml"]
  queue  %[echo "-----> Be sure to edit '#{deploy_to}/#{shared_path}/config/database.yml' and 'secrets.yml'."]

  if repository
    repo_host = repository.split(%r{@|://}).last.split(%r{:|\/}).first
    repo_port = /:([0-9]+)/.match(repository) && /:([0-9]+)/.match(repository)[1] || '22'

    queue %[
      if ! ssh-keygen -H  -F #{repo_host} &>/dev/null; then
        ssh-keyscan -t rsa -p #{repo_port} -H #{repo_host} >> ~/.ssh/known_hosts
      fi
    ]
  end
end

desc "Deploys the current version to the server."
task :deploy => :environment do
  to :before_hook do
    # Put things to run locally before ssh
  end
  deploy do
    # Put things that will set up an empty directory into a fully set-up
    # instance of your project.
    invoke :'git:clone'
    invoke :'deploy:link_shared_paths'
    invoke :'bundle:install'
    invoke :'rails:db_migrate'
    invoke :'rails:assets_precompile'
    invoke :'deploy:cleanup'

    to :launch do
      queue "mkdir -p #{deploy_to}/#{current_path}/tmp/"
      queue "touch #{deploy_to}/#{current_path}/tmp/restart.txt"
    end
  end
end

# For help in making your deploy script, see the Mina documentation:
#
#  - http://nadarei.co/mina
#  - http://nadarei.co/mina/tasks
#  - http://nadarei.co/mina/settings
#  - http://nadarei.co/mina/helpers
```

Biz kendimize göre düzenlemeleri yapmalıyız.

```zsh
set :domain, 'siteadresi.com' # ve ya bir IP adresi olabilir

set :user, 'kullaniciadi' # sunucu da ssh oturumu açmak için kullanıcı
set :port, '22' # ssh portu
set :forward_agent, true # forward_agent kulanılacak ise
set :identity_file, '~/.ssh/id_rsa.pub'  # gerekiyorsa ssh key

set :deploy_to, '/var/www/projem' # sunucuda deploy edilecek path
set :repository, 'git@github.com:user/repo.git' # deploy edilecek repository
set :branch, 'master' # deploy edilecek branch
```

Eğer sunucuda rbenv ve ya rvm kullanılıyorsa deploy.rb dosyasında ilgili satırları uncomment(başındaki diyez) yapmalıyız. Örneğin rbenv kullanılıyorsa;

```zsh
require 'mina/rbenv'
#...
task :environment do
#...
    invoke :'rbenv:load'
#...
end
#...
```

ya da rvm kullanılıyorsa;

```zsh
require 'mina/rvm'

set :rvm_path, '/usr/local/rvm/bin/rvm' # rvm pathi
#...
task :environment do
#...
    invoke :'rvm:use[ruby-1.9.3-p125@default]' # rvm ruby versiyonu
#...
end
#...
```

## Sunucuyu hazırlamak

```zsh
ssh -i ~/.ssh/id_rsa.pub kullaniciadi@xxx.xxx.xxx.xxx # ve ya kullaniciadi@domain.com
mkdir -p /var/www/projem
chown -R kullaniciadi /var/www/projem
```

Diğer tüm ayarlamalar için [buraya](http://nadarei.co/mina/settings/) bakabilirsiniz.

Hazırlık için tüm ayarlamaları yaptığımıza göre şimdi mina dan sunucuyu ayarlamasını isteyebiliriz. Bunun için `mina setup` komutunu çalıştırıyoruz.

İşte hepsi bu kadar, fakat sunucuda oluşturulan dosyalarda database.yml, secret.yml dosyalarını düzenlememiz lazım. Tekrar sunucuya ssh ile girip database ve secret.yml dosyalarını güncelleyebilirsiniz. Bundan sonra görevleri(task) ve komutları öğreneceğiz.

## Görevler

En son mina setup görevini çalıştırmıştık. Bu komut aşağıdaki tanımlanmış görevi çağırdı.

```zsh
task :setup => :environment do
    #.....
end
```

Mina ile bu şekilde ihtiyacımız olan görevleri oluşturabiliriz. Mesela rails loglarını izlemek için ssh ile sunucuya manuel bağlanmadan tail komutu çalıştırabiliriz.

Örnek:
```zsh
task :logs do
  queue 'echo "Contents of the log file are as follows:"'
  queue "tail -f #{deploy_to}/#{shared_path}/log/production.log"
end
```

Şimdi `mina logs` diye komut gönderdiğimizde production log dosyasını izlemeye başladık.

İşte `mina deploy` komutunu çalıştırdığımızda da aslında mina nın yaptığı bu. Ssh ile sunucuya bağlanıyor, ve bir takım ön tanımlı görevleri çalıştırıyor. Bunlar, default olarak sırasıyla

```zsh
#...
invoke :'git:clone'
invoke :'deploy:link_shared_paths'
invoke :'bundle:install'
invoke :'rails:db_create' # bu normalde yoktu ben ekliyorum
invoke :'rails:db_migrate'
invoke :'rails:assets_precompile'
invoke :'deploy:cleanup'

to :launch do
  queue "mkdir -p #{deploy_to}/#{current_path}/tmp/"
  queue "touch #{deploy_to}/#{current_path}/tmp/restart.txt" # passenger ile uygulamayı yeniden başlatıyor
end
#...
```

Daha fazlası için [buraya](http://nadarei.co/mina/about_deploy_rb.html) göz atabilirsiniz.

## [Komut çalıştırmak](http://nadarei.co/mina/tasks/run.html)

Mina nın tek yapabildiği bir kaç görev ile bir projeyi deploy etmek değildir, mesela sunucuya bir ls çekmek için ssh bağlantısı kurup önce cd ile dizine gidip sonra ls çekmeniz gerekir. Mina komutlarınızı yerel bilgisayarınızdan çalıştırıp bu işlemi tek komutta yapmanıza da olanak sağlıyor. Örneğin;

```zsh
mina run[ls]
# ve ya
mina "run[rake log:clear]"
```

Bu yazımda benim gibi capistrano ile uğraşıp da konfigurasyonlarını çok zahmetli bulanlar ve deploy user i ayarlamakta zorlananlar için faydalı bir gem paylaşmayı umdum. Umarım umduğumu bulmuşumdur. Mina hakkında başka soru sormak isteyen olursa yorumlarda elimden geldiğince cevaplarım. Esen kalın.
