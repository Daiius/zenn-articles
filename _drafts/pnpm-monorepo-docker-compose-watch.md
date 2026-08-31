# 執筆計画：pnpm monorepo + docker compose 開発環境を試行錯誤した話

> このファイルは記事執筆用のメモ（計画）です。`articles/` `books/` 以外なので zenn には同期されません。
> public リポジトリのため、**非公開リポジトリ名・内部固有情報は書かない**。コードは複数の自作プロダクトの構成を一般化した**架空例**で示す。

## 方針・トーン
- 目標文字数：**8,000〜10,000 字程度**（短め）。盛り込むより削る。メイン章から書き、随時カウントして調整。
- 姿勢：**「これがベスト」と断定しない**。volume mount 方式から `docker compose watch` 方式へ「better な方向」を**試行錯誤した記録**として書く。
- 軸：before / after を対比し、**メリットだけでなくデメリットも正直に**書く。

## メタ情報（案）
- タイトル：「**pnpm monorepo + docker compose 開発環境を試行錯誤した話**」（過去形。「結論は出たのか？」という関心で読んでもらう狙い。実際、現構成はかなり良い線にいるという手応えがある）
- emoji：🐳 ／ type：tech
- topics：`docker`, `dockercompose`, `pnpm`, `monorepo`, `nextjs`
- published：false（まず下書き）
- 想定読者：pnpm monorepo を Docker compose で開発している人／macOS で HMR・node_modules 肥大に悩んだ人

## 題材にする架空構成（記事全体で共通）
- `nextjs`（フロント）＋ `server-hono`（Hono RPC の API）＋ `database`（Drizzle + MySQL）の 3 パッケージ
- ルートに `pnpm-workspace.yaml`
- 「特定リポジトリの実物ではなく、複数の自作プロダクトの構成を一般化した例」と明記する

---

## 調査結果（手法クオリティ検証｜2026-06 時点）
> 「明らかなデメリットのない、より一般的な上位互換構成はないか」を Web 調査した結論。
- **結論：現構成は現時点のベストプラクティスとほぼ一致。上位互換の"標準構成"は存在しない**。記事の軸として十分。
- 根拠：
  - Docker 公式が watch を bind mount の現代的代替と位置づけ。`sync`/`rebuild`/`sync+restart` の使い分け、node_modules を sync しない理由（小ファイル大量の高 I/O・OS/アーキ差のバイナリ非互換）も公式が明記。
  - 最も引用される monorepo×watch リファレンス（dev.to moofoo）も同じ骨格（watch sync＋node_modules はイメージに焼く＋pnpm store キャッシュ＋lockfile で rebuild）。
  - 対抗馬「bind mount＋node_modules を匿名 volume でマスク」は、まさに逃げたかった欠点（macOS の inotify 取りこぼし・依存 de-sync・root 所有ファイル）を抱える旧来手法。乗り換え方向は妥当。
- **記事に補強する2点**：
  1. 射程の明示：moofoo は Turborepo `prune`＋`dependsOn` で「build/codegen が要る shared パッケージ」を解いている。本記事の構成が単純なのは **Hono RPC が"型のみ共有"でビルド不要**だから。→「codegen 共有パッケージがある場合は watch の restart 対象や turbo の依存設定が別途要る」と一言添える。
  2. 公式制約をデメリット章へ：watch は **`build:` 指定サービスのみ対応（`image:` のプリビルドは非対応）**／`ignore` は監視 `path` 相対／glob 非対応／コンテナに stat・mkdir・rmdir と書込権限が必要。

## 章立て（改訂版・文字数目安つき）

### 1. はじめに（〜600字）
- 複数の個人プロダクトで pnpm monorepo + docker compose 開発環境を運用してきた背景
- 「ベストを提示する記事」ではなく「より良い方向を**試行錯誤**した記録」だと最初に宣言
- 結論先出し：volume mount → compose watch で何が良くなり、何を引き換えにしたか（対比表へ誘導）

### 2. 題材：扱う monorepo の形（〜700字）
- 上記の架空 3 パッケージ構成を簡潔に図示／箇条書き
- Hono RPC で型が server → nextjs に流れる点だけ軽く触れる（後段の watch 設計の伏線）

### 3. Before：volume mount 方式と不満点（〜1,200字）
- 旧 compose 抜粋（架空）：パッケージ別 Dockerfile、`volumes: - ./nextjs:/workspace/nextjs`、匿名 volume の node_modules、`install` one-shot サービス
- 不満点を具体的に：
  - macOS（virtiofs/inotify）で変更取りこぼし → **HMR が不安定**
  - **ホスト node_modules の肥大化**・OS 間バイナリ不整合
  - Dockerfile 乱立＋ install サービスや named volume の**依存チェーンが複雑**

### 4. After：docker compose watch の核（〜3,000字｜記事の山）
要素を 3 点に圧縮して書く（pnpm store キャッシュは Dockerfile.dev のコードで自然に見せる）：
- 4-1. **bind mount を全廃**し `Dockerfile.dev` で `COPY` してイメージに焼く（なぜ HMR が安定し node_modules が肥大しないか）
  - 架空 `Dockerfile.dev` 全文：manifest 先行 COPY → `--mount=type=cache` で pnpm store キャッシュ → `--frozen-lockfile` → ソース COPY
- 4-2. **`develop.watch` の 3 アクション**：`sync`（フロントは HMR 任せ）/ `sync+restart`（tsx サーバ）/ `rebuild`（`pnpm-lock.yaml` 変更時）
  - 架空 `compose.yaml` 抜粋
- 4-3. **env_file をサービス別分割＆ルート集約**＋⚠️罠「watch は bind mount が無いので**サブディレクトリの `.env` はコンテナに届かない**」
- `pnpm dev` = `docker compose watch` に統一（scripts、ローカル node_modules 不要）は短く添える
- 4-4. **射程の注記**（1段落）：この単純さは **Hono RPC が"型のみ共有"でビルド不要**だから成立。codegen/build が要る shared パッケージ（Prisma generate 等）がある場合は watch の restart 対象追加や Turborepo の `dependsOn` 設定が別途必要、と一言断る

### 5. Before / After 対比（〜1,000字）
- 表でメリット／デメリットを並べる
- メリット：HMR 安定、node_modules 非肥大、Dockerfile 集約、依存追加の自動 rebuild、再現性
- **デメリットも明記**：初回イメージ build に時間／`sync` は一方向コピーでファイル削除・リネームに癖／「即編集→即反映」に watch 起動という一手間／コンテナ内 node_modules を IDE から直接覗きにくい
- **公式由来の制約も添える**：watch は `build:` 指定サービスのみ（`image:` プリビルド非対応）／`ignore` は監視 `path` 相対・glob 非対応／コンテナに stat・mkdir・rmdir と書込権限が必要

### 6. まだ迷っている軸（〜1,800字｜試行錯誤の核心）
断定せず「どんな時どちらが better か」で 2〜3 軸だけ：
- 6-1. サーバの `sync+restart` vs `sync` のみ（`tsx watch` に再起動を任せるか）
- 6-2. DB を `tmpfs`（毎回クリーン）vs named volume 永続化（開発データ保持）
- 6-3. DB 初期化：one-shot サービス vs ホストの `pnpm db:migrate`/`db:seed`（冪等化）
- （フロントをコンテナ内 vs ホスト運用、は字数が許せば軽く一段落。厳しければカット）

### 7. まとめ（〜500字）
- 「ベスト」ではなく現時点の better。残課題（初回 build 時間、sync の制約）と今後の探索余地

---

## カット候補（字数が超えたら削る順）
1. 応用トピック（単一イメージ共有 / `compose.override.yaml` / pnpm `catalog` + `minimumReleaseAge`）→ 今回は基本カット。余れば「他にもこんな工夫がある」と数行だけ。
2. ハマりどころ集 → 各章に小さく溶かし込み、独立章にはしない。
3. 6-4（フロントの配置）→ 字数次第でカット。

## 作業ステップ
1. この計画で合意 → 新ブランチ（例：`feature/compose-watch-article`）
2. `articles/pnpm-monorepo-docker-compose-watch.md` に `published: false` で執筆
3. メイン章（3〜5）から書いて文字数を見ながら密度調整 → 1,2,6,7 を埋める
4. レビュー → 公開設定や図・GIF の要否を判断
