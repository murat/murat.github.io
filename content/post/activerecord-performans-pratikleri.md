---
title: "ActiveRecord Performans Pratikleri"
date: 2019-03-12T11:38:34+30:00
authors:
  - Murat Bastas
---

`count`, `present?` ve `where` metodlari yanlis kullanildiginda genellikle N+1 query problemine yol acar.

### .count pratikleri

`count` metodu her zaman sql query atarak count almaya calisir, bu gereksiz sql querysinden kurtulmak icin asagidaki metodlari inceleyebiliriz.

***`size`***

`size` metodu basit bir ruby array countu calistirarak elimizdeki recordset te kac item oldugunu sayar. `size` attigimiz data eger bir active record relation collection'u ise asagidaki implementasyona bakalim.

```zsh
# File activerecord/lib/active_record/relation.rb, line 210
def size
  loaded? ? @records.length : count(:all)
end
```

buna gore `size` metodu eger relation preload edilmis ise `length` ile elimizdeki record seti array item sayar gibi sayar, fakat load edilmemis ise database e count query atarak sorar, yani her daim bize count'u verebilir ve bunu gereksiz sql query atmaktan kacinarak yapar.

ornek:

```zsh
## select * from posts
posts.each do |post|
    puts post.title
end

## count posts array
posts.size
```

yukaridaki ornek database e sadece select query atar

```zsh
## select count(*) from posts
posts.size

## select * from posts
posts.each do |post|
    puts post.title
end
```

yukaridaki ornek database e 1 defa count 1 defa da select atar

```zsh
## select * from posts
posts.load.size

posts.each do |post|
    puts post.title
end
```

yukaridaki ornek database e sadece 1 select atar.

***aksi durum:***

eger postlari iterate ederek gostermeyecek isek, sadece count ise gostermek istedigimiz database e `select * from` query si atmak elbette sacma, bu yuzden ne zaman `count` ne zaman `size` kullanacagimizi bilmeliyiz. ornegin:

```zsh
# select count(*) from posts
Post.all.size

# select * from posts
Post.all.load.size
```

yani elimizdeki activerecord seti load edilmis ise count degil size kullanmaliyiz.

### .where pratikleri

onccelikle activerecord base den turemis her collection .where metodu cagirdigimizda oncelikli olarak database e where query si atmaya calisir. soyle bir senaryomuz oldugunu dusunelim

```zsh
## select * from posts
posts.each do |post|
    ## select count(*) from comments where post_id = ID and published = true
    puts posts.comments.where(published: true).count
end
```

post sayisi kadar comments tablosuna query attik.

```zsh
## select * from posts
## select * from comments where post_id in (ID1,ID2...)
posts.includes(:comments).each do |post|
    ## select count(*) from comments where post_id = ID and published = true
    puts posts.comments.where(published: true).count
end
```

yine olmadi gordugumuz gibi

```zsh
class Post < ActiveRecord::Base
    has_many :comments
    has_many :published_comments, -> { published }
end
class Comment < ActoveRecord::Base
    belongs_to :posts
    scope :published, -> { where.not(published_at: nil) }
end

## select * from posts
## select * from comments where post_id in (ID1,ID2...) and published_at is not null
posts.includes(:published_comments).each do |post|
    ## select count(*) from comments where post_id = ID and published = true
    puts posts.comments.where(published: true).count
end
```
