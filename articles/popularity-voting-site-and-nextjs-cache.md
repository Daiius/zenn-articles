---
title: "Next.js お試し投票サイトのキャッシュ管理で404になった話"
emoji: "🫥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs"]
published: false
publication_name: "chot"
---

# 下書き中
> Next.js で開発しているお試し投票サイトで、
> - `generateStaticParams` でビルド時に集計ページを生成したい
> - `export const dynamicParams = false` でビルド時に生成した集計ページ以外は404にしたい
> - `revalidatePath` を投票時に呼び出して、最小限のページをすぐに更新したい
> 
> という目的で上記の設定や機能を呼び出したところ、
> **`revalidatePath` で更新したページが何度再読み込みしても 404 Not Found エラー表示される** ことに気付きました。
> 
> この現象を記事にしてみたいです。
