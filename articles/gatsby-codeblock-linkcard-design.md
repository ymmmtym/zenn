---
title: "【Gatsby】コードブロックとリンクカードをデザインする"
emoji: "🔮"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["gatsby", "markdown", "blog"]
published: true
---

https://blog.ymmmtym.com/

個人ブログを運用しています。
詳細はブログ作成時に投稿した記事をご覧ください。

[Gatsbyでブログを開設する① 【インストール編】](https://blog.ymmmtym.com/2020/09/20/01)

Zenn や Qiita、はてなブログなどに投稿するほどではない内容をのもの（技術メモなど）は個人ブログに上げています。

個人ブログの主な使用技術は以下の通りです。

- Gatsby.JS
  - `npm` でパッケージ管理
- GitHub
  - 記事も含めてソースコードすべて管理しています。
- Netlify
  - ブログのホスティング先です。

Gatsby にはブログ用のテンプレートが用意されており、ほぼデフォルトのレイアウト設定を使用しておりました。

https://calpa.me/

デフォルトのレイアウトも気に入っておりますが、次のレイアウトを少し変更してみます。

- コードブロック
- リンクカード

## はじめに

`React` 含めてフロントエンドは無知なのですが、これを行うモチベーションは以下になります。

- コードブロックをターミナルっぽくしたい
- 埋め込みリンクはなんとなくかっこいい
  - クリックしたくなってしまう
  - 文字のリンクは少し素っ気ない

## 実装

個人ブログに使用したテンプレートがすでに充実していることもあり、ほんの少しのコード修正で実現できる箇所もありました。

### コードブロックの背景を変更する

コードブロックのレイアウトは、 `Prism.js` で実装されています。

https://prismjs.com/

私が使用したテンプレートにはすでに導入されていたので、そこに記載されていたテーマを変更するだけで問題ありませんでした。

`gatsby-browser.js` で読み込む `css` を `prism-tomorrow` に変更することで、レイアウトを変更できました。

```jsx
import 'prismjs/themes/prism-tomorrow.css';
```

### コードブロックに名前をつける

`gatsby-remark-code-titles` を使用すると、コードブロックに名前が付けられるようです。

https://github.com/otanu/gatsby-remark-prismjs-title

`npm` でインストールします。

```bash
npm install gatsby-remark-prismjs-title
```

インストールできたら、`gatsby-config.json` に記載されている `gatsby-transformer-remark` のプラグインとして読み込むように記述します。

```jsx
    resolve: 'gatsby-transformer-remark',
      options: {
        plugins: [
          {
            resolve: 'gatsby-remark-code-titles',
          },

```

これでコードブロックのタイトルが表示されるようになります。

実際に Markdown で書くときは、`title=<タイトル名>` と記述します。 [^1]

~~~

```javascript:title=title.js
const title = "personal blog"
```

~~~

レイアウトを調整する場合は、適用される `scss` ファイル（`blog-post.scss` など）を修正します。

```scss
pre[class*="language-"] {
  padding: 1em;
  margin-top: 0;
  margin-bottom: 1em;
  overflow: auto;
}

code[class*='language-'],
pre[class*='language-'] {
  color: #fff;
}

:not(pre)>code[class*='language-text'] {
  background-color: #dbdbdb;
  color: #000;
}

.gatsby-code-title {
  background: #462f83;
  color: rgb(255, 255, 255);
  padding: 6px 12px;
  font-size: 0.8em;
  line-height: 1;
  font-weight: bold;
  display: table;
}
```

### コードブロックをコピーできるようにする

`gatsby-remark-code-buttons` を使用すると、コードブロックをコピーできます。

https://github.com/iamskok/gatsby-remark-code-buttons

`npm` でインストールします。

```bash
npm install gatsby-remark-code-buttons
```

インストールできたら、`gatsby-config.json` に記載されている `gatsby-transformer-remark` のプラグインとして読み込むように記述します。

```jsx
      resolve: 'gatsby-transformer-remark',
      options: {
        plugins: [
          {
            resolve: 'gatsby-remark-code-buttons',
            options: {
              toasterText: 'Copied!!',
            },
          },
```

これで、コードブロックを 1 クリックでコピーできます。

レイアウトを調整する場合は、適用される `scss` ファイル（`blog-post.scss` など）を修正します。

```scss
.gatsby-code-button {
  position: absolute;
  top: -17px ;
  right: 10px;
  z-index: 100;
  &::after {
      display: none !important;
  }
  svg {
      filter: invert(98%) sepia(5%) saturate(983%) hue-rotate(178deg)
          brightness(95%) contrast(99%);
      opacity: 0.9;
  }
}
```

### リンクカードを設定する

想像以上に実装が難しかったです。

まずは、`@raae/gatsby-remark-oembed` を使用します。[^2]

https://github.com/raae/gatsby-remark-oembed

`npm` でインストールします。

```bash
npm install @raae/gatsby-remark-oembed
```

Twitter が表示できることは確認できましたが、表示できるサイトが限定されているようです。
対応しているサイトは以下より確認ができます。

https://oembed.com/providers.json

対応していないサイトであっても、oembed API が公開されていれば、`gatsby-config.json` に設定を追加することで表示できます。

たとえば、はてなブログは oembed API が公開されています。

上述した、oembed.com に登録されていない場合でも、`settings` に記載することで表示できます。

```jsx
          {
            resolve: '@raae/gatsby-remark-oembed',
            options: {
              usePrefix: false,
              providers: {
                settings: {
                  hatenablog: {
                    endpoints: [
                      {
                        schemes: ['https://*.hatenablog.com/*'],
                        url: 'https://hatenablog.com/oembed',
                      },
                    ],
                  },
                },
              },
            },
          },
```

ある程度のサイトであれば、埋め込みで表示できました。

目標は URL を貼り付けただけで埋め込みリンクが表示されることなので、他を探してみます。

https://iframely.com/embed

これは、URL ベタばりではなく、一度 iframely と言うサイトで埋め込み `html` コードを取得して markdown に貼り付けます。
ほぼすべてのサイトを埋め込みリンクで表示できます。

実装は簡単で、`Helmet` 部分に以下のように iframely の cdn を読み込むように設定します。[^3]

```jsx
<Helmet>
  <script type="text/javascript" src="https://cdn.iframe.ly/embed.js" />
<Helmet/>
```

目標は URL を貼り付けただけで表示されることでしたが、使用できるサイトには `gatsby-remark-oembed` を使って、それ以外は `html` を書くように整理しています。

## さいごに

コードブロックは好みな感じになりました。

リンクカードはすべてのサイトに対応できてないので、次回は `component` 部分を改修してよりコアな所に触れていきます。

## Reference（参考文献）

コードブロックについて。

https://kikunantoka.com/2019/12/03--install-syntax-highlight/

https://nekoniki.com/blog/gatsby-syntax-highlight/

https://blog.wadackel.me/2020/hugo-to-gatsby/

https://kimuson.dev/blog/gatsby/gatsby-blog/

https://saruwakakun.com/html-css/reference/border

リンクカードについて。

https://blog.leko.jp/post/gatsby-remark-discoverable-oembed/

https://www.luku.work/gatsby-link-card

https://blog.petitviolet.net/post/2020-03-06/oembed-expansion

https://expfrom.me/gatsby-use-iframely.md/

[^1]:  `title=<タイトル名>` の `title=` を省略しても表示されるようにしたいです。
[^2]: markdown 記法である `[title](URL)` 形式も埋め込み表示されてしまう問題点？もあります。
[^3]: 環境によっては初回アクセス時にうまく読み込まれないことがあるので、`Helmet` 側で  `useEffect` を使って `iframely` コンポーネントがある場合に読み込むなどの工夫が必要です。
