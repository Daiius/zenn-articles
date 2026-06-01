---
title: "tmux で coding agents の入力待ちペインに移動する話"
emoji: "🔔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["claudecode", "tmux", "rust", "cli", "codexcli"]
published: false
---

## 概要

Claude Code や Codex CLI のような、coding agents を **tmux のあちこちのペインで同時に走らせていると、「どこかで誰かが返事を待っている」状態になりがち**です。

権限要求のダイアログが出ているペイン、ターンが終わって次のプロンプトを待っているペイン、長いビルドが終わったまま放置されているペイン……。

これを解決するために **bellmux** という小さな CLI を作りました。

@[card](https://github.com/Daiius/bellmux)

## どう使うのか - 
(スクリーンキャプチャ準備中)

## 機能
tmux と coding agents の間を繋ぐ **最小限の機能を意識しています**。

1. **イベント通知**：
  「このペインで待ち状態になったよ」という、通知を記録する機能
2. **イベント解除**
  「このペインの待ち状態が解除されたよ」という、該当する通知を削除する機能
3. **イベント有無返却**
  「待ち状態のペインがあるよ」という、単純な読み取り機能
4. **通知の来た pane\_id を 1 つずつ返却**
  「次の待ち状態のペインはこれだよ」という、順序の付いた読み取り機能

tmux には run-shell コマンドが、coding agents には hooks があります。
適切なタイミングでスクリプトを実行する仕組みが既に用意されているため、実装するべき機能は少ないです。
:::details tmux: run-shell の具体例
```bash
# tmux はキーバインドで任意のコマンドを実行する様に設定できます
bind-key a run-shell '任意のコマンド・スクリプト'
```
:::
:::details Claude Code: hooks の具体例
```json
// Claude Code / Codex CLI どちらも hook という仕組みで、様々なイベント発生時にスクリプトを実行できます。
"Notification": [ // ← Stop, UserPromptSubmit, PostToolUse など様々
  {
    "matcher": "",
    "hooks": [
      {
        "type": "command",
        "command": "任意のコマンド・スクリプト"
      }
    ]
  }
],
```
:::


```mermaid
sequenceDiagram
  participant tmux as tmux
  participant bellmux as 「何か」
  participant hooks as Claude Code hooks

  note over tmux,hooks: シナリオ① : 待ち状態で tmux 表示変更
  alt 入力待ち発生時
    hooks ->> bellmux : イベント通知
  else 待ち解除時
    hooks ->> bellmux : 解除通知
  end
  loop status-interval で<br>定期的に動作 
    tmux ->> bellmux : 通知有無など取得
    bellmux -->> tmux : 
    tmux -->> tmux : ステータスバー等の<br>表示を変更
  end
```

```mermaid
sequenceDiagram
  participant tmux as tmux
  participant bellmux as 「何か」
  note over tmux,bellmux: シナリオ② :  待ち状態のペインに移動
  tmux -->> tmux : キーバインド実行
  tmux ->> bellmux : 待ち状態ペイン要求
  bellmux -->> bellmux : 複数待ち状態なら cyclic に返却<br>無効 $pane_id は削除して次へ
  bellmux -->> tmux : $pane_id 返却<br>(1 周したことも判別可能に)
  tmux -->> tmux : 待ち状態の pane にジャンプ<br>（もし 1 周したらその旨 status bar に表示）
  

```

## tmux との連携設定
興味のある方向けに、tmux と coding agent の設定内容をもう少し詳しく示しています。

:::details tmux キーバインド

`bellmux init --preset keybinds` で出力される、未対応ペイン巡回用のキーバインドです。

```sh
# 次の未対応ペインへジャンプ
bind-key a run-shell '
  read -r pane tag <<<"$(bellmux next)"
  if [ -z "$pane" ]; then
    tmux display-message "No pending notifications"
    exit 0
  fi
  tmux switch-client -t "$pane"
  if [ "$tag" = wrapped ]; then
    tmux display-message "Cycled through all pending notifications."
  fi
'

# 手動 ack
bind-key A run-shell 'bellmux ack-pane --pane-id "#{pane_id}" && tmux refresh-client -S'
bind-key X run-shell 'bellmux ack-all && tmux refresh-client -S'
```

| キー | 動作 |
|---|---|
| `prefix + a` | 次の未対応ペインへジャンプ（古い方向へ巡回） |
| `prefix + b` | 前の未対応ペインへジャンプ（新しい方向へ巡回） |
| `prefix + A` | 現在ペインの通知を全 ack |
| `prefix + X` | 全 ack |

**`bellmux next` 自身は ack しません**。「見に行っただけかもしれない」ので、ack は明示操作（`UserPromptSubmit` / `PostToolUse` / `PostToolUseFailure` / `prefix + A` など）でしか起きません。
:::

:::details Claude Code hooks 設定

bellmux の最初のモチベーションは Claude Code との協調だったので、`~/.claude/settings.json` に貼る hook が一番磨かれています。`bellmux init --preset claude-hooks` で出力されるのはこちら。

```json
{
  "hooks": {
    "Notification": [{
      "matcher": "permission_prompt|elicitation_dialog",
      "hooks": [{"type": "command", "command": "bellmux push --kind notification --pane-id \"$TMUX_PANE\" && bellmux bell"}]
    }],
    "Stop": [{
      "matcher": "",
      "hooks": [{"type": "command", "command": "bellmux push --kind stop --pane-id \"$TMUX_PANE\" && bellmux bell"}]
    }],
    "UserPromptSubmit": [{
      "matcher": "",
      "hooks": [{"type": "command", "command": "bellmux ack-pane --pane-id \"$TMUX_PANE\""}]
    }],
    "PostToolUse": [{
      "matcher": "",
      "hooks": [{"type": "command", "command": "bellmux ack-pane --pane-id \"$TMUX_PANE\""}]
    }],
    "PostToolUseFailure": [{
      "matcher": "",
      "hooks": [{"type": "command", "command": "bellmux ack-pane --pane-id \"$TMUX_PANE\""}]
    }],
    "SessionEnd": [{
      "matcher": "",
      "hooks": [{"type": "command", "command": "bellmux ack-pane --pane-id \"$TMUX_PANE\""}]
    }]
  }
}
```

各 hook の対応は次の通りです。

| Hook | タイミング | bellmux 動作 |
|---|---|---|
| Notification | 権限要求 / MCP 入力要求 (elicitation) | `push --kind notification && bell` |
| Stop | ターン完了 | `push --kind stop && bell` |
| UserPromptSubmit | 新しいプロンプト送信 | `ack-pane` |
| PostToolUse | ツール実行完了（成功） | `ack-pane` |
| PostToolUseFailure | ツール実行失敗 | `ack-pane` |
| SessionEnd | セッション終了 | `ack-pane` |

**ポイントは `Notification` の matcher です。** `permission_prompt|elicitation_dialog`（権限ダイアログ / MCP サーバーの入力要求）に一致したときだけ hook が発火し、idle ping などはそもそも bellmux に届きません。**「どの通知を拾うべきか」の判断を hook 設定側に寄せた**結果、bellmux 本体は coding agent の通知種別（`notification_type`）を一切知りません。surface 対象を増やしたいときは matcher に `|` 区切りで足すだけで、bellmux は無変更です。
:::

:::details Codex CLI hooks 設定

Codex CLI も同じ要領で連携できます。設定はユーザースコープの `~/.codex/hooks.json` に置きます（Claude Code の `settings.json` とは別ファイル）。`bellmux init --preset codex-hooks` で出力されるのはこちら。

```json
{
  "hooks": {
    "PermissionRequest": [{
      "matcher": "",
      "hooks": [{"type": "command", "command": "printf '%s' '{\"message\":\"Codex needs approval\"}' | bellmux push --kind notification --pane-id \"$TMUX_PANE\" && bellmux bell"}]
    }],
    "Stop": [{
      "matcher": "",
      "hooks": [{"type": "command", "command": "printf '%s' '{\"message\":\"Codex turn complete\"}' | bellmux push --kind stop --pane-id \"$TMUX_PANE\" && bellmux bell"}]
    }],
    "UserPromptSubmit": [{
      "hooks": [{"type": "command", "command": "bellmux ack-pane --pane-id \"$TMUX_PANE\""}]
    }],
    "PostToolUse": [{
      "matcher": "",
      "hooks": [{"type": "command", "command": "bellmux ack-pane --pane-id \"$TMUX_PANE\""}]
    }],
    "SessionStart": [{
      "matcher": "startup|resume|clear",
      "hooks": [{"type": "command", "command": "bellmux ack-pane --pane-id \"$TMUX_PANE\""}]
    }]
  }
}
```

各 hook の対応は次の通りです。

| Hook | タイミング | bellmux 動作 |
|---|---|---|
| PermissionRequest | コマンド承認要求 | `push --kind notification && bell` |
| Stop | ターン完了 | `push --kind stop && bell` |
| UserPromptSubmit | 新しいプロンプト送信 | `ack-pane` |
| PostToolUse | ツール実行完了 | `ack-pane` |
| SessionStart | セッション開始 / 再開 / clear | `ack-pane` |

**Claude Code との差分は「イベント名」と「matcher の要否」だけ**です。

- 承認要求のイベントが `Notification` ではなく `PermissionRequest` で、これは承認要求専用イベントなので matcher で種別を絞る必要がなく `""` のままで済みます。
- セッション系は `SessionEnd` ではなく `SessionStart`（`startup|resume|clear`）で、起動・再開時に古い通知を掃除します。
- 通知メッセージは `printf '%s' '{...}' | bellmux push` と固定 JSON を stdin で流しているだけです。bellmux 側はやはり通知種別を知らないので、**どちらの agent でも push / ack-pane という同じ語彙**で繋がります。
:::

## tmux / coding agents 以外の組み合わせもOK
様々なツールと組み合わせて使用可能です。

- **特定の coding agent に依存しません**
  - 受け取った通知をそのまま記録するだけで、「どれを拾うか」は前述の通り hook の matcher 任せです（Codex 用プリセットも固定 JSON を流すだけ）
  - なんなら shell script 実行さえできれば任意のツールから bellmux を呼び出せます
- **Linux / macOS / Windows でも OK**
  - 実際にすべてのプラットフォームで動作させた訳ではないですが、原理的には幅広い OS で動作するはずです
  - 私は OS の音を鳴らすコマンドを使用しています、OS ごとに異なるコマンドでも環境に合わせて設定できます
- **Zellij や screen にも対応（のはず）** 
  - snippet を追加するだけ、Rust 側は無変更です
  - screen は画面管理の単位が若干異なるため、完全に同じようには動作しないかもしれません

## おわりに
「 tmux で複数の coding agents を同時に走らせる時、入力待ちペインにジャンプしたい」という課題の解決策は色々ありそうです。同じ課題にアプローチするもの、似ている発想のものも割とあります。
@[card](https://github.com/shuntaka9576/agentoast)
@[card](https://zenn.dev/kki2ne/articles/claude-code-hooks-macos-notification-2026)
@[card](https://lib.rs/crates/tmux-claude-queue)
bellmux は「特定の OS / coding agents に依存していない」ことが特徴になっていそうです。色々なアプローチがある中で、1 つ面白い例になっているといいなと思います。
