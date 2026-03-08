---
title: "秘密にしたい URL が Google 検索に載ってしまった話"
emoji: "🔎"
type: "tech"
topics: ["google", "seo", "hono", "vite", "searchconsole"]
published: false
---

## はじめに — 何が起きたのか

サカバンバスピスの顔をランダム生成する Web アプリを開発しました。seed 値からキャラクターの顔を生成し、気に入った顔を `?seed=xxx` 付きの URL でシェアできます。

顔がサカバンバスピスっぽいかで名前もサカバンバスピスっぽくなったり、ならなかったりします。


メインの遊び方として、中々出ない **「サカバンバスピス」が出るまで顔生成ボタンを連打してもらうというのがあります** 。
ですが、**見落としに気づきました**。


ある日、自分のサイト名で Google 検索してみたところ、**ユーザーがシェアした「サカバンバスピス」seed 付きの URL が検索結果に表示されている**ことに気づきました。


:::message 
もちろん、**こうやってユーザーに URL をシェアしてもらうのはとても嬉しいです！**

ですが検索結果からアクセスできてしまうと特に、
**本来自分で発見するのが醍醐味の「サカバンバスピス」を簡単に発見できるシステムになっており、楽しみを奪ってしまう**
という課題が明らかになりました。
:::

この記事では、この失敗に気づいてから対応するまでの経緯を紹介します。

### スクショなど

@[card](https://sacapis2026.faveo-systema.net)

| | |
|---|---|
|![](/images/sacapis/screenshot-02.png)|![](/images/sacapis/screenshot-03.png)|

## なぜインデックスされたのか

**根本的な原因は、`canonical` を設定していなかったことです。**
全体的な経緯や影響を整理すると、次のような状況でした。

```mermaid
sequenceDiagram
    participant User as ユーザー
    participant SNS as SNS
    participant Bot as Googlebot
    participant Google as Google 検索

    User->>SNS: seed 付き URL をシェア
    Bot->>SNS: リンクを発見・クロール
    Bot->>Google: ページをインデックスに追加
    Google->>Google: 検索結果に表示
```

このアプリは OGP 画像をシード毎に描画するため、Hono で SSR（サーバーサイドレンダリング）しています。Googlebot にとっては各 seed 付き URL が**実質的に別のページ**に見えていたわけです。

:::message
Next.js 等のフレームワークであれば、HTML のキャッシュ制御や SEO 関連の設定がある程度カバーされますが、Hono + Vite の軽量構成ではこういった部分を自分で対応する必要があります。自由度が高い反面、設定漏れのリスクがあることを痛感しました。
:::

## 対応その1: Google Search Console の所有権確認

まず、Search Console でサイトを管理できるようにするため、所有権の確認を行いました。確認方法はいくつかありますが、今回は HTML メタタグを埋め込む方法を選びました。

@[card](https://support.google.com/webmasters/answer/9008080?hl=ja)

```tsx
{/* Google Search Console Tags */}
<meta
  name="google-site-verification"
  content="（google search console で指示されたメタタグの値）"
/>
```

Hono の SSR テンプレート（`IndexPage.tsx`）の `<head>` 内に上記のタグを追加しました。

## 対応その2: Google Search Console から検索結果を削除

所有権の確認ができたら、既にインデックスされてしまった URL を検索結果から消すために、Google Search Console の「削除」機能を使いました。

@[card](https://developers.google.com/search/docs/crawling-indexing/remove-information?hl=ja)


Google Search Console には「一時的な削除」という機能があり、特定の URL やプレフィックスに一致する URL を約6ヶ月間、検索結果から非表示にすることができます。

:::message alert
この機能はあくまで**一時的な非表示**です。6ヶ月後には再びインデックスされる可能性があるため、根本的な対策（canonical の設定）が別途必要です。
:::

## 対応その3: canonical URL の設定

すべてのページに `canonical` タグを追加し、正規 URL をトップページに統一しました。これにより、seed 付きページはトップページの重複コンテンツとして扱われ、検索結果にはトップページのみが表示されるようになります。

```tsx
// app.tsx — URL の生成
const canonicalUrl = baseUrl.endsWith("/") ? baseUrl : `${baseUrl}/`;
```

```tsx
// IndexPage.tsx — canonical タグの出力
<link rel="canonical" href={canonicalUrl} />
```

ここでのポイントは、**`og:url` と `canonical` で異なる URL を使い分けている**点です。

| 用途 | URL | 例 |
|------|-----|----|
| `og:url`（SNS シェア用） | seed 付き | `https://sacapis2026.faveo-systema.net/?seed=12345` |
| `canonical`（SEO 用） | seed なし | `https://sacapis2026.faveo-systema.net/` |

`og:url` に seed 付き URL を使うことで、SNS でシェアされたときに正しい OGP 画像が表示されます。一方、`canonical` はルート URL を指すことで、Google に「これらのページの正規版はトップページです」と伝えています。

## 考察: noindex ではなく canonical を選んだ理由

実は最初の対応では、seed ページに `noindex` と `canonical` を**両方**設定していました。

```tsx
{/* 当初の実装 */}
{seed && <meta name="robots" content="noindex, follow" />}
<link rel="canonical" href={canonicalUrl} />
```

しかし、これは**混合シグナル**になりうることに気づきました。

- **`noindex`** = 「このページをインデックスしないで」
- **`canonical`（他ページへの参照）** = 「このページの正規版は別の URL にある」

両方を同時に指定すると、Google にとっては意図が曖昧になります。Google の公式ドキュメントによると、この場合は `noindex` が優先される傾向があるようですが、そもそも矛盾したシグナルを送ること自体が望ましくありません。

### noindex と canonical の比較

| アプローチ | メリット | デメリット |
|-----------|---------|-----------|
| `noindex` + `canonical` | 確実に検索結果から除外 | 混合シグナル、SEO シグナルの統合が不十分 |
| `canonical` のみ | セマンティックに正しい、SEO シグナルが統合される | canonical は「ヒント」であり Google が独自に判断する可能性がある |

### 最終的な判断

最終的に **`noindex` を外して `canonical` のみ**にしました。理由は以下の通りです。

- canonical をルートに向ければ、Google は seed ページをルートの重複コンテンツと見なし、検索結果にはルート URL のみを表示する — これが今回やりたいことそのもの
- `canonical` はあくまで「ヒント」であり Google が無視する可能性はゼロではないが、同一コンテンツの URL パラメータ違いという今回のケースは canonical が最も尊重されやすい典型的なパターン
- SEO シグナル（被リンク等）がトップページに統合されるため、サイト全体の検索順位にもプラスに働く

## robots.txt について

補足として、`robots.txt` についても触れておきます。現状では seed 付き URL のクロール抑制をコメントアウトしています。

```txt
User-agent: *
Allow: /

# seed付きURLへのクロールを抑制（noindex効果確認後に有効化）
# Disallow: /*?seed=
```

canonical の効果が Search Console 上で確認できたら、`Disallow` を有効化する予定です。ただし、**`robots.txt` の `Disallow` はクロール自体を抑制するものであり、既にインデックスされた URL の削除には効果がない**点には注意が必要です。既にインデックスされた URL を消すには、Search Console の削除機能や canonical による正規化が必要です。


## おまけ：アプリ側の対応
:::details シェアされた顔の区別

SEO 対策とは別に、アプリ側でも対応を行いました。

前述の通り、このアプリには「サカバンバスピスを自分で見つける楽しみ」があります。
しかし、検索結果から seed 付き URL にアクセスしたユーザーには、**それが自分で発見した顔なのか、他人がシェアした顔なのか区別がつかない**という問題がありました。

そこで URL の seed に対応する名前が図鑑機能で発見済みでない場合、「シェアされたかお」として表示する仕組みを追加しました。

これにより、検索やシェアから来たユーザーも、そこから自分自身の探索を始められるようになっています。
:::

## まとめ

今回の失敗から得た教訓をまとめます。

- **公開前に `canonical` 設定を一通り見直すべき**。特にクエリパラメータで内容が変わるページは、意図せずインデックスされるリスクがある
- **フレームワークに頼らず SSR する場合、SEO 関連の設定は自分で責任を持つ必要がある**。Hono + Vite のような軽量構成では、こうした設定を忘れがち
- **Google Search Console は早めにセットアップしておく**。問題が発覚してからでは、所有権確認から始めることになり、対応が遅れる

些細な設定漏れでしたが、実際に起きてみると意外と焦りました。同じような構成でアプリを作っている方の参考になれば幸いです。
