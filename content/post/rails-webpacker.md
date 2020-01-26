---
title: Rails webpacker ile vuejs
date: 2017-07-30T12:42:31+30:00
authors:
  - Murat Bastas
---

Öncelikle - [Rails 5.1 Loving JavaScript](http://weblog.rubyonrails.org/2017/4/27/Rails-5-1-final/)

Merhaba yukarıda linkini verdiğim yazıda da görebileceğiniz gibi Rails 5.1 versiyonundan itibaren [rails/webpacker](https://github.com/rails/webpacker) aracılığıyla projelerimizi React, Angular, Elm ve ya Vuejs yazabileceğimiz şekilde oluşturma imkanına sahibiz. Bu yazımda vuejs ile basit bir proje oluşturarak öğrendiklerimi anlatacağım. Aklımda örnek proje olarak github api v3 kullanarak repo arayan basit bir uygulama yapmak var.

# Projeyi oluşturalım...

```zsh
rails new webpacker-vue-example --webpack=vue -B
cd webpacker-vue-example
```

Kısa şekilde anlatmak gerekirse —webpack=vue ile projemizde vuejs kullanacağımızı söyledik ve webpacker bizim için gerekli tüm konfigurasyonları yaptı. Proje dizininde `npm install` ve ya `yarn install` diyerek webpacker'ın oluşturduğu `package.json` dosyasındaki bağımlılıkları kurduk.

```zsh
bundle install
rails g controller root home --skip-helper --skip-assets
```

Örnek uygulamamızın giriş sayfası için root adında bir controller oluşturduk. Bu giriş sayfasında repo aramamızı sağlayan bir autocomplete componenti olacak. `config/routes.rb` dosyamıza uygulamamızın evini söyleyelim.

```zsh
Rails.application.routes.draw do
  root 'root#home'
end
```

# Packimizi hazırlayalım...

Rails uygulamamızın app dizininde `javascript/packs` diye bir dizin var. Packlerimiz burada duracak. Örnek olarak **hello_world** packini inceleyebilirsiniz. Ben pack dizinimi aşağıdaki şekile getiriyorum.

```zsh
tree app/javascript
app/javascript
├── app.sass
└── packs
    ├── App.vue
    ├── Card.vue
    └── app.js
```

`packs/app.js` dosyamızda aşağıdaki gibi vue instance oluşturuyoruz.

```js
import Vue from 'vue/dist/vue.esm'
import App from './App.vue'
import '../app' // sass dosyamız

Vue.config.productionTip = false;

document.addEventListener('DOMContentLoaded', () => {
    /* eslint-disable no-new */
    new Vue({
        el: '#app',
        template: '<App/>',
        components: { App }
    })
})
```

Vue **#app** elementine mount edilecek ve App.vue dosyasındaki template'i render edecek. App componentimiz bir autocomplete ve bir de card componenti render edecek. Autocomplete remote search ile github v3 api den repositories endpointine request atıp repository listesini çekecek ve seçtiğimiz repository card componenti ile render edilecek.

## Autocomplete için element ui kullanacağım...

```zsh
yarn add element-ui
yarn add babel-plugin-component -D
```

Element UI içinde tonla component barındıran bir UI paketi. Ama biz sadece autocomplete componentini kullanacağımız için **.babelrc** dosyamızda plugins e aşağıdaki eklemeyi yapıyoruz.

```js
{
...
    "plugins": [
        ...
        ["component", [
            {
                "libraryName": "element-ui",
                "styleLibraryName": "theme-default"
            }
        ]]
        ...
    ]
...
}
```

`packs/app.js` dosyamıza da aşağıdakileri ekleyerek element ui den istediğimiz componentleri import ediyoruz.

```js
...
import { Autocomplete } from 'element-ui'
...
Vue.component(Autocomplete.name, Autocomplete)
...
```

## Request göndermek için axios kullanacağım..

```zsh
yarn add axios
```

`packs/App.vue` dosyama aşağıdaki gibi bir template yazıyorum.

```vue
<template>
    <div id="app">
        <el-autocomplete v-model="value"
                    @select="select"
                    :fetch-suggestions="querySearch"
                    placeholder="Please enter a keyword">
        </el-autocomplete>

        <card v-if="selected" :repo="selected"></card>
    </div>
</template>

<script>
    import axios from 'axios'
    import Card from './card'

    export default {
        name: 'App',

        components: { Card },

        data() {
            return {
                value: '',
                repositories: [],
                selected: null
            }
        },

        methods: {
            querySearch(queryString, cb) {
                const that = this

                this.repositories = []

                if (queryString) {
                    axios.get(`https://api.github.com/search/repositories?sort=name&order=asc&q=${queryString}`)
                    .then((response) => {
                        response.data.items.map((v) => {
                            return Object.assign(v, {
                                value: v.name,
                                label: v.id,
                            })
                        }).forEach((v) => {
                            that.repositories.push(v)
                            cb(that.repositories)
                        })
                    })
                }
            },

            select(item) {
                this.selected = this.repositories.filter(v => v.id === item.label)
            }
        }
    }
</script>
```

Ve card template'imizi yazalım.

```vue
<template>
    <div class="card">
        <div class="item" v-for="item in repo">
            <h3><a :href="item.owner.html_url" target="_blank">{{item.owner.login}}</a> / <a :href="item.html_url" target="_blank">{{item.name}}</a></h3>

            <p>{{item.description}}</p>

            <span class="meta" v-if="item.language">
                <span class="language-icon" style="color: #4C5E99;">●</span> {{item.language}}
            </span>
            <span class="meta">
                <icon name="star"></icon>
                <span>
                    {{item.stargazers_count}}
                </span>
            </span>
            <span class="meta">
                <icon name="code-fork"></icon>
                <span>
                    {{item.forks_count}}
                </span>
            </span>
        </div>
    </div>
</template>

<script>
    export default {
        name: 'Card',
        props: ['repo']
    }
</script>
```

#  Ve artık çalıştırabiliriz...

Fakat packlerin ve rails server ın aynı anda çalışıyor olması gerek o yüzden bir sekmede `rails s` bir sekmede `./bin/webpack-dev-server` çalışmalı. Bu işi [foreman](https://github.com/ddollar/foreman) ile kolaylıştırabiliriz. Bu yüzden foreman kurduktan sonra( `gem install foreman` ) proje dizininde bir `Procfile` oluşturup içine aşağıdakileri yazıyoruz.

```Procfile
web: bundle exec rails s
webpacker: ./bin/webpack-dev-server
```

Şimdi proje dizininde;

```zsh
foreman start
```

Ve [http://localhost:5000](http://localhost:5000) üzerinden projenin durumunu görebilirsiniz.

**Dipnot:**

Sorusu olan yoruma, projeyi klonlayıp incelemek isteyen [github reposuna](https://github.com/muratbsts/rails-webpacker-vuejs) buyursun. Kalın sağlıcakla ✋
