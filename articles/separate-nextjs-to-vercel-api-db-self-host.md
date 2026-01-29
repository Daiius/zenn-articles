---
title: "Hono RPC で Next.js と API サーバを型安全に分離する"
emoji: "🔗"
type: "tech"
topics: ["hono", "nextjs", "typescript"]
published: false
---

## はじめに

Next.js の Server Actions や Server Components は、フロントエンドからサーバーサイドの処理を直接呼び出せる便利な機能です。しかし、この利便性の裏には「フロントエンドとサーバーロジックが密結合になる」という構造的な特性があります。

2025 年末に公表された React Server Components の重大な脆弱性 CVE-2025-55182 は、未認証のリモートコード実行（RCE）を可能にするものとして注目を浴びました。この事例が示すのは、密結合の構造そのものが想定外のセキュリティリスクを生む可能性です。

この記事では、明示的な BFF（Backend for Frontend）層を設けて API エンドポイントとして扱う構成を、Hono RPC を使って型安全に実現する方法を紹介します。

## モチベーション

### RSC の利便性と潜在的リスク

Server Actions や Server Components を使うと、フロントエンドのコードから直接サーバーサイドの関数を呼び出せます。API エンドポイントを明示的に定義する必要がなく、開発体験は非常にスムーズです。

しかし、この構造には注意点があります。フロントエンドとサーバーロジックが直接結びついているため、フレームワーク側に脆弱性があった場合、その影響がサーバーサイドの処理全体に及ぶ可能性があります。

2025 年末に公表された CVE-2025-55182 は、RSC の処理に起因する未認証のリモートコード実行（RCE）脆弱性でした。Next.js を含む RSC をサポートするフレームワークが影響対象となり、IPA などから注意喚起が出されました。

### BFF 層を設けるメリット

明示的な BFF 層を設けて API エンドポイントとして扱う構成には、以下のメリットがあります。

**影響範囲の限定**

HTTP エントリポイントを意図的に設計することで、どの操作が外部から呼び出し可能かを明確にコントロールできます。フレームワークの脆弱性があっても、BFF 層で許可された操作しか実行できないため、影響範囲を限定できます。

**セキュリティ制御の一元化**

BFF 層に WAF（Web Application Firewall）、認証、レート制限などのセキュリティ制御を集約できます。

- WAF: 攻撃パターンの検知・遮断
- 認証: JWT、API キーなど柔軟な認証方式
- レート制限: 異常なリクエスト頻度を制限
- ログ監視: HTTP アクセスログによる監視

**権限の機能単位での分離**

API 経由であれば、権限を「機能単位」に分離できます。

- 参照専用 API と更新用 API を分ける
- 特定のリソースのみアクセス可能にする
- 認証が突破されても、API で許可された操作しかできない

### 注意: この構成のトレードオフ

この構成では、Vercel 側は Serverless でスケールしますが、BFF（API サーバ）は単一サーバで動作するため、ここがボトルネックになります。スケーラビリティの観点では理想的な構成ではありません。

以下のような状況であれば、この構成でも問題になりにくいです。

- バックエンドへのアクセス頻度が低く、処理が軽量
- 負荷集中時のレイテンシ増加（スケールしない BFF 部分がボトルネックになる分）が許容できる
- BFF 自体を Serverless 構成（Cloud Run、Fly.io など）で運用している

## Hono RPC とは

Hono RPC は、サーバ側のルート定義から型情報をエクスポートし、クライアント側で型安全な API クライアントを生成する仕組みです。

特徴:
- サーバ側でルートを定義すると、その型情報を `AppType` としてエクスポートできる
- クライアント側で `hc<AppType>()` を呼ぶと、パスやパラメータ、レスポンスの型が自動補完される
- OpenAPI スキーマの生成やコード生成は不要

## API 設計の実践

BFF 層を設計する際に意識したポイントを紹介します。

### エンドポイント設計

リソース指向で設計し、HTTP メソッドで操作を表現します。

| エンドポイント | メソッド | 操作 |
|----------------|----------|------|
| `/characters` | GET | キャラクター一覧を取得 |
| `/votes/:id` | GET | 特定ユーザーの投票を取得 |
| `/votes/:id` | POST | 投票を登録・更新 |

ポイント:
- パスはリソースを表し、メソッドで操作を区別
- パスパラメータ (`:id`) でリソースを特定
- 一覧取得と個別取得を明確に分離

### 返却データの設計

レスポンスは型安全性を意識して設計します。Hono RPC では `c.json()` の戻り値から型が推論されるため、一貫した構造にしておくとクライアント側で扱いやすくなります。

### Zod バリデーション

`@hono/zod-validator` を使うことで、リクエストパラメータやボディの検証と型推論を同時に行えます。

- パスパラメータ: `zValidator('param', z.object({ id: z.string() }))`
- リクエストボディ: `zValidator('json', z.array(...))`
- クエリパラメータ: `zValidator('query', z.object(...))`

### 認証ミドルウェア

ミドルウェアで認証処理を一元化することで、各エンドポイントに認証ロジックを書く必要がなくなります。

- Bearer Token を使った API キー認証
- 認証失敗時は早期に 401 を返す
- 認証成功時のみ `await next()` で次の処理へ

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


## まとめ

Hono RPC を使うことで、以下のメリットが得られました。

- **型安全**: パス、パラメータ、リクエストボディ、レスポンスすべてに型補完が効く
- **コード生成不要**: OpenAPI スキーマの生成や codegen なしで型が共有できる
- **セキュリティ境界の明確化**: BFF 層で HTTP エントリポイントを意図的に設計
- **制御の一元化**: WAF・認証・レート制限を BFF 層に集約

RSC の利便性を享受しつつも、セキュリティの観点からフロントエンドとバックエンドを分離したい場合に、Hono RPC は良い選択肢だと思います。
