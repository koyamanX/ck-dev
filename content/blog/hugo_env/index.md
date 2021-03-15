---
title: "Getting started with hugo"
date: 2021-03-15T06:40:17+09:00
author: "@koyamanX"
categories: ["hugo"]
tags: ["tutorial", "setup", "hugo"]
draft: false
---

## Hugoを使ってみる.
### モチベーション
今までは、紙ベースのノートに情報をまとめていたが、
- 検索ができない
- 画像貼り付けがめんどうくさい
- コピペができない

など不便な点がいくつかあった。\
そこで、データとして情報を集約していきたいと考えた。
いろいろ調べているうちに、hugoが簡単そうであったため導入してみた。
Markdownで記述ができ、静的ページに変換する仕組みのようである。\
以下に導入と簡単な使用方法をまとめておく。

<!--more-->

### インストール
環境としてUbuntu 20.04.2 LTSを仮定する。

[hugo公式インストールガイド](https://gohugo.io/getting-started/installing)
[hugo公式Quick Start](https://gohugo.io/getting-started/quick-start/)

#### hugoインストール
```bash
sudo apt install hugo
```

#### サイトの作成
```bash
hugo new site ck-dev
```

#### テーマの追加
デザインがいい感じなので、テーマとしてnotepadiumを採用する。
[Notepadium Github](https://github.com/cntrump/hugo-notepadium)

```bash
cd ck-dev
git init
git submodule add https://github.com/cntrump/hugo-notepadium.git themes/hugo-notepadium
```

#### テーマの設定
```toml
baseURL = "http://localhost/"
languageCode = "en-us"
title = "ck-dev"
theme = "hugo-notepadium"
copyright = "©2021 koyamanX"

[markup.highlight]
codeFences = true
noClasses = false

[markup.goldmark.renderer]
unsafe = true  # enable raw HTML in Markdown

[params]
style = "auto"  # default: auto. light: light theme, dark: dark theme, auto: based on system.
dateFormat = "Monday, January 2, 2006"  # if unset, default is "2006-01-02"
logo = ""  # if you have a logo png
slogan = "Notes for myself"
license = ""  # CC License
fullRss = false # Puts entire HTML post into rss 'description' tag. If unset, default is false.

[params.comments]
enable = true  # En/Disable comments globally, default: false. You can always enable comments on per page.

[params.math]
enable = false  # optional: true, false. Enable globally, default: false. You can always enable math on per page.
use = "katex"  # option: "katex", "mathjax". default: "katex"

[params.syntax]
use = "none"  # builtin: "prismjs", "hljs". "none" means Chroma
theme = "xcode"
darkTheme = "xcode-dark"  # apply this theme in dark mode

[params.share]
enable = true
addThisId = "x-1234567890"
inlineToolId = ""

[params.nav]
showCategories = true       # /categories/
showTags = true             # /tags/

# custom navigation items

[[params.nav.custom]]
title = "About"
url = "/about"

```

#### ポストの作成

```bash
hugo new blog/first-post.md
```

`contents/blog/first-post.md`が作成される。

ファイルを開き、編集する。
```toml
---
title: "Getting started with hugo"
date: 2021-03-15T06:40:17+09:00
author: "@koyamanX"
categories: ["hugo"]
tags: [""]
draft: true
---
# 本文
```
著者、カテゴリー、タグを追加する。
毎回追加するのはめんどうくさいので、`archetypes/default.md`へ
```toml
---
title: "{{ replace .Name "-" " " | title }}"
date: {{ .Date }}
author: "@koyamanX"
categories: ["hugo"]
tags: [""]
draft: true
---
```
を記載しておく。
`hugo new`でページを作成したときに使われるようになる。

`---`の下に本文をMarkdownにて記述する。
いい感じに`<!--more-->`を一つ入れることで、カテゴリーなどで表示したときに見やすくなる。

なお、draft: trueとなっているのは、投稿がドラフト段階にあることを意味しており、静的ページに変換する際に、オプションが必要である。

```bash
hugo	# without Draft
hugo -D	# with Draft
```

#### 確認
```bash
hugo server
```

localhost:1313にアクセスしてページが表示されれば成功。

#### Nginxをウェブサーバーとして使う。
hugoには組込みのウェブサーバーがあるらしい。
[hugo server](https://gohugo.io/commands/hugo_server/)
適当にページを見る分にはこれでよさそうであるが、将来公開する用(現状はローカルホストで自分用)に一応hugo+Nginxで環境を構築してみる。
[Deploying a static Hugo site with NGINX](https://gideonwolfe.com/posts/sysadmin/hugonginx/)
```bash
sudo apt install nginx
sudo cp /etc/nginx/sites-available/default /etc/nginx/sites-available/hugo
sudo ln -s /etc/nginx/sites-available/hugo /etc/nginx/sites-enabled/hugo
```

`/etc/nginx/sites-available/hugo`を開き、
rootをhugoのサイトにする。
```
servers {
	...
	root /home/koyaman/ck-dev;
	...
}
```

#### Nginxの起動
```bash
sudo systemctl enable nginx
sudo systemctl start nginx
```

`localhost`へアクセスし、ページが見れることを確認する。

いい感じ

