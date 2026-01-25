---
title: "Hono RPC で Next.js と API サーバを型安全に分離する"
emoji: "🔗"
type: "tech"
topics: ["hono", "nextjs", "typescript"]
published: false
---

## はじめに

Next.js で Web アプリを作っていると、Server Actions や Server Components から直接データベースを操作することが多いと思います。しかし、プロジェクトが成長するにつれて、以下のような課題が出てくることがあります。

- **責務の分離**: フロントエンドとデータベース操作ロジックが密結合になりがち
- **インフラの制約**: Vercel からセルフホストの DB に安全かつ安定して接続するのが難しい

この記事では、Hono RPC を使って Next.js から API サーバを分離し、型安全な API 連携を実現した方法を紹介します。

:::message
一般的には PlanetScale や Neon などの DBaaS を使うのがシンプルでスケーラブルな構成です。この記事で紹介する構成は、DB をセルフホストしている場合に API サーバを挟んで解決するアプローチです。
:::

## モチベーション

### Next.js から直接 DB を触らないようにする

Server Actions や Server Components で直接 DB アクセスするとコードが密結合になります。DB 操作ロジックを API サーバに集約することで責務を分離できます。

### Vercel から DB に接続する際の課題

技術的には Vercel から セルフホストの MySQL に直接接続すること自体は可能です。しかし、安全かつ安定して運用するのは難しいです。

**IP 制限の困難さ**

DB を守る一般的な方法として IP 制限がありますが、Vercel の Serverless / Edge Functions は実行元 IP が固定されていません。リクエストごとに異なるリージョンや IP から接続される可能性があり、ファイアウォールで特定 IP のみを許可するという防御が使えません。

**コネクション数の問題**

Serverless 環境では、リクエストごとに新しい関数インスタンスが起動します。各インスタンスが DB に接続すると、同時リクエスト数がそのまま同時 DB 接続数になります。MySQL の `max_connections` はデフォルトで 151 程度なので、少しアクセスが集中するとすぐに上限に達してしまいます。

これらの課題を解決するため、API サーバ（BFF）を DB と同じ場所にセルフホストし、Vercel からは API 経由でアクセスする構成にしました。

```
Vercel (Next.js) → API サーバ (Hono) → DB (MySQL)
                   ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                   同一ネットワーク内でセルフホスト
```

### API サーバを挟むメリット

API サーバを挟むことで、セキュリティ・運用・設計の各面でメリットがあります。

#### セキュリティ: プロトコルレベルの違い

MySQL はもともとインターネット公開を前提として設計されたプロトコルではありません。一方、HTTP は公開前提で発展してきたため、多層防御のエコシステムが充実しています。

- **WAF（Web Application Firewall）**: SQL インジェクションなどの攻撃パターンを検知・遮断
- **Rate Limiting**: 異常なリクエスト頻度を制限
- **認証・認可ミドルウェア**: JWT、OAuth など柔軟な認証方式
- **IDS / ログ監視**: HTTP アクセスログは解析ツールが豊富

MySQL をインターネットに公開すると、これらの防御層を使えません。

- ブルートフォース攻撃、既知の脆弱性（CVE）を突かれるリスク
- MySQL のデフォルトポート 3306 はスキャンされやすい

MySQL に TLS を設定すれば通信経路の盗聴や改ざんは防げますが、それはあくまで「通信の安全性」を確保するものであり、「MySQL をインターネットに公開すること自体のリスク」を解消するものではありません。特に Serverless 環境からの直接接続ではコネクション管理や DoS 耐性の問題もあり、実運用では API サーバを経由させてデータベースを内部ネットワークに閉じ込める構成がより安全で安定した設計になります。

#### セキュリティ: 認証突破時のリスク差

MySQL の認証が突破された場合、攻撃者は SQL を直接実行できます。`SELECT * FROM users` で全ユーザー情報を抜いたり、`DROP TABLE` でデータを破壊したりできてしまいます。

API 経由であれば、権限を「機能単位」に分離できます。

- 参照専用 API と更新用 API を分ける
- 特定のテーブルやカラムのみアクセス可能にする
- 認証が突破されても、API で許可された操作しかできない

#### 運用: Serverless とコネクションプール

前述のコネクション問題の解決策として、API サーバが「流量制御レイヤー」として機能します。

- API サーバは常駐プロセスなので、コネクションプールを維持できる
- 複数のリクエストが来ても、プール内の接続を使い回す
- DB への同時接続数を一定に保てる

#### 設計: 3層アーキテクチャの思想

クラウド設計では、DB をプライベートサブネットに配置するのが基本です。

```
パブリック        プライベート
[Web サーバ] → [API サーバ] → [DB]
```

今回の構成はこれと同じ思想です。API サーバを挟むことで DB は内部ネットワークに隠せます。DB は `localhost` または Docker 内部ネットワークでのみ接続可能にし、ファイアウォールで 3306 を閉じられます。

### 補足: API を挟まなくても良いケース

DB を直接公開することが問題なのではなく、「パブリックインターネットに MySQL を直接公開する」ことが問題です。以下のケースでは API を挟まなくても安全に運用できます。

- **社内 VPN 経由**: インターネットに露出していないため、IP 制限や認証で十分守れる
- **クラウドの PrivateLink / VPC Peering**: DB がプライベートネットワーク内にあり、パブリックに公開されない
- **PlanetScale / Neon などの DBaaS**: 公開前提で設計されたプロキシ層があり、コネクション管理や認証が適切に行われている

今回のようにセルフホストの MySQL を Vercel から使う場合は、API サーバを挟むのが現実的な解決策です。

### 注意: この構成のトレードオフ

この構成では、Vercel 側は Serverless でスケールしますが、BFF（API サーバ）は単一サーバで動作するため、ここがボトルネックになります。スケーラビリティの観点では理想的な構成ではありません。

以下のような状況であれば、この構成でも問題になりにくいです。

- バックエンドへのアクセス頻度が低く、処理が軽量
- 負荷集中時のレイテンシ増加（スケールしない BFF 部分がボトルネックになる分）が許容できる
- BFF 自体を Serverless 構成（Cloud Run、Fly.io など）で運用している

## Before / After

### Before: Next.js 内で直接 DB アクセス

`next/src/lib/` 内で Drizzle ORM を使って直接 DB 操作していました。

| ファイル | 関数 | 処理内容 |
|----------|------|----------|
| `characters.ts` | `getCharacters()` | キャラクター一覧を取得 |
| `votes.ts` | `getLatestVotes(twitterID)` | ユーザーの最新投票 |
| `votes.ts` | `insertVotesIfUpdated()` | 投票が変わっていれば INSERT |
| `votes.ts` | `getLatestVotesForAnalysis()` | 全キャラの投票分析 |
| `votes.ts` | `getTimelineData(charaName)` | 30日間の履歴トレンド |

```
Next.js Pages/Components
    ↓ Server Actions / Server Components
lib/characters.ts, lib/votes.ts（Drizzle ORM で直接 DB 操作）
    ↓
MySQL Database
```

### After: Hono API サーバに分離

| エンドポイント | メソッド | 対応する元の関数 |
|----------------|----------|------------------|
| `/characters` | GET | `getCharacters()` |
| `/analysis` | GET | `getLatestVotesForAnalysis()` |
| `/analysis/:charaName` | GET | 特定キャラの投票分析 |
| `/timeline/:charaName` | GET | `getTimelineData(charaName)` |
| `/votes/:id` | GET | `getLatestVotes(twitterID)` |
| `/votes/:id` | POST | `insertVotesIfUpdated()` |

```
Next.js (Vercel)
    ↓ HTTP + Bearer Token
Hono API サーバ（セルフホスト）
    ↓ Drizzle ORM
MySQL Database（セルフホスト）
```

## Hono RPC とは

Hono RPC は、サーバ側のルート定義から型情報をエクスポートし、クライアント側で型安全な API クライアントを生成する仕組みです。

特徴:
- サーバ側でルートを定義すると、その型情報を `AppType` としてエクスポートできる
- クライアント側で `hc<AppType>()` を呼ぶと、パスやパラメータ、レスポンスの型が自動補完される
- OpenAPI スキーマの生成やコード生成は不要

## 実装例

### サーバ側: ルート定義と型エクスポート

```typescript:server-ts/src/app.ts
import { Hono } from 'hono'
import { zValidator } from '@hono/zod-validator'
import { z } from 'zod/v4'
import { getCharacters } from './lib/characters'
import { getLatestVotes, insertVotesIfUpdated } from './lib/votes'

const apiKey = process.env.API_KEY ??
  (() => { throw new Error('process.env.API_KEY is not defined!') })()

export const app = new Hono()

// 単純なAPIキー認証
app.use('*', async (c, next) => {
  const authHeader = c.req.header('Authorization')
  if (!authHeader) return c.body(null, 401)
  const tokens = authHeader.split(' ')
  if (tokens.length != 2) return c.body(null, 400)
  const [bearer, apiKeyToken] = tokens
  if (bearer !== 'Bearer') return c.body(null, 400)
  if (apiKeyToken !== apiKey) return c.body(null, 401)
  await next()
})

const route = app
  .get('/characters', async (c) => {
    const characters = await getCharacters()
    return c.json(characters, 200)
  })
  .get(
    '/analysis/:charaName',
    zValidator('param', z.object({ charaName: z.string() })),
    async c => {
      const { charaName } = c.req.valid('param')
      const data = await getLatestVotesForAnalysis(charaName)
      return c.json(data, 200)
    }
  )
  .get(
    '/votes/:id',
    zValidator('param', z.object({ id: z.string() })),
    async c => {
      const { id } = c.req.valid('param')
      const data = await getLatestVotes(id)
      return c.json(data, 200)
    },
  )
  .post(
    '/votes/:id',
    zValidator('param', z.object({ id: z.string() })),
    zValidator('json', z.array(z.object({
      characterName: z.string(),
      level: z.number(),
    }))),
    async c => {
      const { id } = c.req.valid('param')
      const votes = c.req.valid('json')
      const updated = await insertVotesIfUpdated({ twitterID: id, data: votes })
      // 変更されたキャラ名を返す（Next.js側でrevalidatePathに使う）
      return c.json(updated, 200)
    },
  )

// RPC用の型情報をエクスポート
export type AppType = typeof route

// クライアント用のhcもre-export
export { hc } from 'hono/client'
```

ポイント:
- `const route = app.get(...).post(...)` のようにメソッドチェーンでルートを定義
- `typeof route` で型情報を取り出し、`AppType` としてエクスポート
- `hc` もパッケージから re-export しておくと、クライアント側で import しやすい

### クライアント側: API クライアントの生成

```typescript:next/src/lib/apiClient.ts
import { hc, type AppType } from '@daiius/girls-side-analysis-server-ts'

const apiKey = process.env.API_KEY
  ?? (() => { throw new Error('process.env.API_KEY is not defined') })()
const apiUrl = process.env.API_URL
  ?? (() => { throw new Error('process.env.API_URL is not defined') })()

const createCustomedFetch = (
  options?: NextFetchRequestConfig
): typeof fetch => async (
  input: RequestInfo | URL,
  requestInit?: RequestInit,
) => {
  const headers = new Headers(requestInit?.headers)
  headers.set('Authorization', `Bearer ${apiKey}`)
  return await fetch(input, { ...requestInit, headers, next: options })
}

export const client = (
  options?: NextFetchRequestConfig
) => hc<AppType>(apiUrl, { fetch: createCustomedFetch(options) })
```

ポイント:
- `hc<AppType>(apiUrl)` で型安全なクライアントを生成
- カスタム `fetch` を渡すことで、認証ヘッダの付与や Next.js の `revalidate` オプションを設定できる

### 使用例: 型補完が効く

```typescript:next/src/lib/characters.ts
import { client } from './apiClient'

export const getCharacters = async () => {
  // revalidate: 86400 で1日キャッシュ
  const res = await client({ revalidate: 86400 }).characters.$get()
  if (res.ok) {
    return await res.json() // 戻り値の型が推論される
  }
  throw new Error(`cannot fetch characters: ${res.status}`)
}
```

```typescript:next/src/lib/votes.ts
import { client } from './apiClient'

export const getLatestVotes = async (twitterID: string) => {
  // パスパラメータも型チェックされる
  const res = await client().votes[':id'].$get({ param: { id: twitterID } })
  if (res.ok) {
    return await res.json()
  }
  throw new Error(`cannot fetch votes: ${res.status}`)
}

export const insertVotesIfUpdated = async ({
  twitterID,
  data,
}: {
  twitterID: string
  data: Vote[]
}) => {
  // POST のリクエストボディも型チェックされる
  const res = await client().votes[':id'].$post({
    param: { id: twitterID },
    json: data, // { characterName: string, level: number }[] が期待される
  })
  if (!res.ok) {
    throw new Error(`cannot post votes: ${res.status}`)
  }
  const { updatedCharaNames } = await res.json()
  // 変更されたキャラのページだけ revalidate
  for (const charaName of updatedCharaNames) {
    revalidatePath(`/${encodeURIComponent(charaName)}`)
  }
}
```

## モノレポでの型共有

サーバ側の `AppType` を Next.js から参照するために、pnpm workspace でモノレポ構成にしました。

### pnpm-workspace.yaml

```yaml:pnpm-workspace.yaml
packages:
  - 'next'
  - 'server-ts'
```

### server-ts/package.json

```json:server-ts/package.json
{
  "name": "@daiius/girls-side-analysis-server-ts",
  "type": "module",
  "exports": {
    ".": {
      "import": "./src/app.ts"
    }
  },
  "dependencies": {
    "hono": "^4.8.10",
    // ...
  }
}
```

### next/package.json

```json:next/package.json
{
  "devDependencies": {
    "@daiius/girls-side-analysis-server-ts": "workspace:*"
  }
}
```

ポイント:
- `workspace:*` で同一ワークスペース内のパッケージを参照
- `devDependencies` に入れることで、型情報のみを使用（実行時コードは含まれない）
- Next.js のビルド時に API サーバのコードはバンドルされない

## おまけ: Vercel デプロイ時の調整

### standalone ビルドの削除

セルフホストしないなら `next.config.ts` の `output: "standalone"` は不要です。Vercel にデプロイする場合は削除しました。

### DB 関連依存の削除

API サーバに DB 操作を移動したので、Next.js 側からは以下の依存を削除できました。

- `drizzle-orm`
- `mysql2`
- `drizzle-kit`

これにより Next.js のビルドがシンプルになり、デプロイサイズも削減できました。

## まとめ

Hono RPC を使うことで、以下のメリットが得られました。

- **型安全**: パス、パラメータ、リクエストボディ、レスポンスすべてに型補完が効く
- **コード生成不要**: OpenAPI スキーマの生成や codegen なしで型が共有できる
- **責務分離**: DB 操作ロジックを API サーバに集約
- **インフラ柔軟性**: Vercel (Next.js) + セルフホスト (API + DB) の構成が可能

セルフホストの DB を使いつつ Vercel にデプロイしたい場合や、フロントエンドと API サーバを分離したい場合に、Hono RPC は良い選択肢だと思います。
