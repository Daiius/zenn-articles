---
title: "macOS 標準機能だけで Secure Enclave に SSH 鍵を置く（調査メモ）"
emoji: "🔐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["macos", "ssh", "security", "secureenclave", "openssh"]
published: false
---

:::message alert
**これは下書きです。** 記事本文ではなく、調査結果を素材としてそのまま置いてあります。
後から構成し直して記事にする前提。
:::

詳細な逆アセンブル記録・検証プログラム全文は
`~/sources/claude/macos-secure-enclave-ssh-investigation.md` にある。
この下書きは「記事にするならどう並べるか」という視点で素材を整理したもの。

---

## 構成案

1. 導入 — macOS 26 Tahoe から、追加インストールなしで Secure Enclave 製 SSH 鍵が使えるようになった
2. 基本手順 — `sc_auth create-ctk-identity` → `ssh-keygen -K` → `~/.ssh/config`
3. **躓きポイント：複数鍵を作ると全部同じファイル名になる**（ここが記事の山場）
4. 原因を逆アセンブルで確定する
5. 確定版の取り出し手順（tmux 2 ペイン）
6. 出回っている誤情報：`KEYCHAIN_CERTIFICATES` は効かない
7. 他の選択肢との比較（Secretive / YubiKey / 1Password / パスフレーズ+Keychain）
8. おまけ：Apple 同梱 ssh は第三者製 provider を dlopen できる
9. 限界と注意点

想定読者：Mac で SSH 鍵の扱いに気を遣いたいが、常駐アプリは増やしたくない人。

---

## 1. 導入に使う話

- 秘密鍵をディスクに置きたくない、という動機
- macOS 26 Tahoe で `/usr/lib/ssh-keychain.dylib` が **SecurityKeyProvider** を実装した
  → OS が Secure Enclave を FIDO2 デバイスのように扱えるようになった
- これまで同じことをするには Secretive などの常駐アプリが必要だった
- 英語圏でもまだ Gist 1 本 + HN 1 スレ + ブログ数本という状況（§9 に一覧）

環境:

```console
$ sw_vers
ProductVersion:  26.6.2
BuildVersion:    25G83
$ ssh -V
OpenSSH_10.3p1, LibreSSL 3.3.6
```

---

## 2. 基本手順（記事の前半に置く）

```bash
# 鍵を作る（-k p-256-ne = 非エクスポータブル、-t bio = Touch ID 必須）
sc_auth create-ctk-identity -l mac-to-worker -k p-256-ne -t bio -N mac-to-worker

# 確認
sc_auth list-ctk-identities
sc_auth list-ctk-identities -t ssh   # ← SSH 形式のフィンガープリントが出る

# credential file として書き出す
cd ~/.ssh
ssh-keygen -w /usr/lib/ssh-keychain.dylib -K -N ""
```

```sshconfig
Host worker
  HostName ...
  IdentityFile ~/.ssh/mac-to-worker
  SecurityKeyProvider /usr/lib/ssh-keychain.dylib
  IdentitiesOnly yes
```

強調しておきたい点：

- 書き出される `id_ecdsa_sk_rk` は **秘密鍵ではなく、Enclave 内の鍵を指すハンドル**
- なので `-N ""`（パスフレーズなし）で問題ない
- ファイルを盗まれても、その Mac の Enclave と Touch ID が無ければ署名できない
- 鍵の形式は `sk-ecdsa-sha2-nistp256@openssh.com` → サーバ側に **OpenSSH 8.2+** が必要

---

## 3. 躓きポイント：複数鍵が全部同じ名前になる

2 本目の identity を作ってから `-K` すると、こうなる:

```console
$ ssh-keygen -w /usr/lib/ssh-keychain.dylib -K -N ""
Enter PIN for authenticator:
You may need to touch your authenticator to authorize key download.
Saved ECDSA-SK key to id_ecdsa_sk_rk
id_ecdsa_sk_rk already exists.
Overwrite (y/n)?
```

全鍵が `id_ecdsa_sk_rk` に書かれようとして衝突する。
`-K` は resident key を**まとめて**取得する機能で、
特定の鍵だけを指定するオプションは OpenSSH 側にも存在しない。

:::message
この衝突自体は英語圏では既知。
[arianvp の Gist](https://gist.github.com/arianvp/5f59f1783e3eaf1a2d4cd8e952bb4acf)
に「application と user_id が設定されていない」として書かれている。
記事では新規性を主張せず、**原因の確定と、実用的な回避手順**を価値として出す。
:::

---

## 4. 原因（逆アセンブルの結果）

### 4-1. `sk_load_resident_keys` は全 token 鍵を無条件に返す

`SecItemCopyMatching` を 1 回呼ぶだけ。環境変数も plist も読まない。

```
kSecClass             = kSecClassKey
kSecAttrKeyClass      = kSecAttrKeyClassPrivate
kSecAttrKeyType       = kSecAttrKeyTypeECSECPrimeRandom
kSecAttrAccessGroup   = kSecAttrAccessGroupToken
kSecAttrKeySizeInBits = 256
kSecMatchLimit        = kSecMatchLimitAll     ← 全件
```

### 4-2. `application` が `"ssh:"` にハードコードされている

```asm
0000000000001648  adrp  x0, 11 ; 0xc000
000000000000164c  add   x0, x0, #0x220        ; __cfstring[0]
0000000000001650  bl    _objc_msgSend$UTF8String
0000000000001654  bl    _strdup
000000000000165c  str   x0, [x19, #0x10]      ; srk->application = strdup(...)
```

`__cfstring[0]` の実体は `"ssh:"`（len=4）。全鍵に同じリテラルが入る。

### 4-3. OpenSSH 側の命名規則

`ssh-keygen` のバイナリに書式文字列が存在する:

```console
$ strings -a /usr/bin/ssh-keygen | grep -E 'id_%s_rk'
id_%s_rk%s%s
```

`do_download_sk()` は application の `"ssh:"` 以降を接尾辞として使う:

```c
xasprintf(&path, "id_%s_rk%s%s",
    key->type == KEY_ECDSA_SK ? "ecdsa_sk" : "ed25519_sk",
    *(srk->application + 4) == '\0' ? "" : "_",
    srk->application + 4);
```

`"ssh:"` の後ろが空 → 接尾辞なし → **全部 `id_ecdsa_sk_rk`**。

本来なら `-O application=ssh:mac-to-worker` で `id_ecdsa_sk_rk_mac-to-worker` になるはずだが……

### 4-4. `sk_enroll` は未実装なのでその逃げ道も無い

```asm
_sk_enroll:
0000000000000f38  mov  w0, #-0x2      ; SSH_SK_ERR_UNSUPPORTED
0000000000000f3c  ret
```

`ssh-keygen -t ecdsa-sk` での鍵生成が拒否される。
`sc_auth create-ctk-identity` を使うしかなく、そちらに application を渡す術はない。**詰み。**

---

## 5. 確定版の取り出し手順

### 5-1. まず「どの鍵か」を順序に頼らず確定できることを示す

`sc_auth list-ctk-identities -t ssh` が出す値は
`ssh-keygen -lf *.pub` の出力と**一致する**（自前で SHA-256 を計算して確認済み）。

```console
$ sc_auth list-ctk-identities -t ssh
p-256-ne SHA256:yNPvOyQ8NyueupRaoS5IEXLBTEZoWENmWfYNRtt0TVA bio mac-to-vps
p-256-ne SHA256:g03eyg3b1MQUBiDqQ5o+bg3e5wQHgcBIGaZTO3Lpz5s bio mac-to-worker
```

なお `ssh-keygen -K` の書き出し順（= `SecItemCopyMatching` の返却順、作成順）と
`sc_auth` の表示順（label のアルファベット順）は**一致しない**ので、順序に頼ってはいけない。

### 5-2. `Overwrite?` 中の `mv` は安全（実験済み）

プロンプト到達時点で、ファイルには**1 つ前に書かれた鍵**が完全な状態で入っている。
待機中に `mv` で退避しても壊れず、`y` を送れば新規作成される（`cmp` でバイト一致を確認）。

### 5-3. 手順

毎回 `mv` してから常に `y`。**1 パス、Touch ID 1 回**で全鍵を回収できる。

```
[ペイン A] cd ~/.ssh && rm -f id_ecdsa_sk_rk id_ecdsa_sk_rk.pub
           ssh-keygen -w /usr/lib/ssh-keychain.dylib -K -N ""
             ↓ "Overwrite (y/n)?" で停止

[ペイン B] ssh-keygen -lf ~/.ssh/id_ecdsa_sk_rk.pub    # 何が入っているか確認
           mv ~/.ssh/id_ecdsa_sk_rk     ~/.ssh/mac-to-worker
           mv ~/.ssh/id_ecdsa_sk_rk.pub ~/.ssh/mac-to-worker.pub

[ペイン A] y                                           ← 鍵の数だけ繰り返す
             ↓ 最後の鍵はプロンプトが出ずに終了する

[ペイン B] ssh-keygen -lf ~/.ssh/id_ecdsa_sk_rk.pub    # 最後の1本を確認してリネーム
```

:::message
必ず**別ペイン（別シェル）**で操作すること。同じシェルに打つと stdin を `ssh-keygen` に食われる。
`mv` は `y` を送る前に完了させる。`ssh-keygen` は `read()` でブロックしているので急ぐ必要はない。
:::

代替案として「プロンプトを N 番目を選ぶスイッチとして使う」方法もある
（全部 `n` → 1 本目、`y` を k-1 回 → k 番目）。
ただし実行のたびに Touch ID が必要。

---

## 6. 出回っている誤情報：`KEYCHAIN_CERTIFICATES` は効かない

`man 8 ssh-keychain` には公開鍵ハッシュで identity を絞れると書いてある。
しかし man の日付は **February 10, 2020** で、PKCS#11 / RSA スマートカード向けの記述。

呼び出しグラフで確定できる:

```
_getCertificateFilter  ← 呼び出し元は _getAvailableSlots のみ
_getAvailableSlots     ← 呼び出し元は _C_Initialize のみ（= PKCS#11 のエントリ）
```

`sk_load_resident_keys` からの到達経路が存在しない。
そもそも p-256-ne 鍵は PKCS#11 経路では扱えない:

```console
$ ssh-keygen -D /usr/lib/ssh-keychain.dylib
cannot read public key from pkcs11
```

:::message
文書が薄い領域なので、LLM に聞くと 2020 年の man page の記述が
新しい sk 経路にも当てはまるものとして混ざりやすい。
自分もこれで一度ハマった、という話を書くと読者の役に立つはず。
:::

---

## 7. 他の選択肢との比較

| 手法 | 秘密鍵の所在 | 可搬性 | 鍵の形式 | 追加依存 | Touch ID が承認するもの |
|---|---|---|---|---|---|
| 本手法 | Secure Enclave | Mac 固定 | `sk-ecdsa-...@openssh.com` | **なし** | 署名 1 回 |
| Secretive | Secure Enclave | Mac 固定 | `ecdsa-sha2-nistp256` | GUI アプリ + 常駐 | 署名 1 回 |
| YubiKey (FIDO2 sk) | YubiKey | ポータブル | `sk-ssh-ed25519@...` | provider dylib | タッチ + PIN |
| YubiKey (PIV) | YubiKey | ポータブル | 標準 RSA/ECDSA | ykcs11 / OpenSC | PIN |
| TPM 2.0 | TPM | マシン固定 | 標準 RSA/ECDSA | 常駐 agent | PIN |
| 1Password SSH Agent | 保管庫 | 同期される | 標準形式 | アプリ + 契約 | Touch ID |
| パスフレーズ + Keychain | **ディスク上** | コピー可能 | 標準形式 | なし | **Keychain の解錠** |

強調したい 2 点:

**(a) 最下段だけ Touch ID の意味が違う。**
`ssh-add --apple-use-keychain` の Touch ID は「パスフレーズを取り出すための Keychain 解錠」で、
一度通れば秘密鍵は agent に平文で常駐する。上の 6 つは「署名 1 回の承認」。
だから **ssh-agent のキャッシュは Secure Enclave 鍵には効かない**（後述）。

**(b) YubiKey は 2 本買って両方登録できるが、Secure Enclave では不可能。**
紛失・故障時の復旧手段が無いのは本手法の明確な弱点。正直に書く。

---

## 8. おまけ：Apple 同梱 ssh は第三者製 provider を dlopen できる

「macOS 標準 OpenSSH は FIDO 未対応、Homebrew 必須」という記述をよく見るが、正確には：

- 内蔵 FIDO（libfido2 / `sk-usbhid`）は**無い**（`strings` で確認、該当文字列なし）
- しかし `ssh-sk-helper` は library validation を解除する entitlement を持つ

```console
$ codesign -d --entitlements - /usr/libexec/ssh-sk-helper
[Key] com.apple.private.security.clear-library-validation
[Value] [Bool] true
```

ad-hoc 署名しただけの自作スタブ dylib を `/private/tmp` から読ませたところ、普通にロードされた:

```console
$ ssh-keygen -w "$PWD/fakeprov.dylib" -K < /dev/null
Enter PIN for authenticator: No keys to download
```

→ **YubiKey を使うのに ssh 本体を差し替える必要はなく、`libsk-libfido2.dylib` だけあればよいはず。**

```sshconfig
Host github.com
  SecurityKeyProvider /opt/homebrew/lib/libsk-libfido2.dylib
```

:::message alert
**未検証。** 手元に YubiKey が無いため、provider がロードされることまでしか確認していない。
記事に載せるなら実機で署名まで確かめてからにする。
:::

---

## 9. 限界と注意点（記事の締め）

### 認証頻度は ssh-agent では下がらない

`sk_sign` が呼ぶ Security API は `SecItemCopyMatching` と `SecKeyCreateSignature` の 2 つだけで、
`LAContext` も `kSecUseAuthenticationContext` も使っていない。
署名のたびに Enclave の ACL が独立に評価されるので、キャッシュする余地が実装上存在しない。
（PKCS#11 側の `C_Login` は `LAContext` をセッションに保持している。sk 経路には移植されていない）

頻度を下げたいなら OpenSSH 標準の接続多重化を使う。追加依存なし。

```sshconfig
Host worker
  ControlMaster auto
  ControlPath ~/.ssh/cm/%C
  ControlPersist 1h
```

公開鍵認証の署名は接続あたり 1 回なので、多重化すれば最初の 1 回だけになる。
代償として、その間は Touch ID なしで接続できる状態がローカルに残る。

### attestation の質は本物の FIDO トークンより低い

- `sk_sign` が返す flags バイトは **ssh から渡された引数をそのまま書き戻している**
  （`mov x21, x6` → `strb w21, [x27]`）。認証器が独立に主張した値ではない
- 署名カウンタは `__bss` 上のプロセスローカル変数。
  `ssh-sk-helper` は ssh 実行ごとに新規プロセスなので、毎回 0 → 1。
  FIDO 本来のクローン検知としては機能しない

Enclave の ACL が署名操作自体を守っているので
「認証なしに署名は存在しない」保証は保たれる。
失われているのは**サーバ側から独立に検証できる attestation**。
物理トークンを別デバイスとして信頼するモデルではなく、「この Mac を信頼する」モデル。

### まだ枯れていない

`ssh-keychain.dylib` が SecurityKeyProvider を実装したのは macOS 26 Tahoe から。
2025 年秋に登場した、まだ 1 年経っていない手法。

英語の一次情報:

- <https://gist.github.com/arianvp/5f59f1783e3eaf1a2d4cd8e952bb4acf>
- <https://news.ycombinator.com/item?id=46025721>
- <https://ewpratten.com/blog/ssh-secure-enclave/>
- <https://gatezh.com/posts/macos-secure-enclave-ssh-keys/>
- <https://algustionesa.com/native-secure-enclave-ssh-keys-the-macos-guide/>
- <https://korben.info/en/protect-ssh-keys-touch-id-macos.html>
- <https://lists.mindrot.org/pipermail/openssh-unix-dev/2024-July/041451.html>

---

## 記事化時の TODO

- [ ] `sc_auth create-ctk-identity` の実行例を実際の出力込みで撮り直す
- [ ] `authorized_keys` の `verify-required` が機能するか検証（`sk_flags` に UV ビットが立つか）
- [ ] YubiKey を入手して §8 の署名経路を検証（できなければ「未検証」と明記して載せる）
- [ ] Apple へのフィードバック（FB21992630 として報告済みという情報があるが未確認）
- [ ] 逆アセンブルの節をどこまで載せるか判断。深すぎると読者が離れるので、
      結論と再現コマンドを主にして、詳細は折りたたむか別記事に分ける
- [ ] タイトル再考。「標準機能だけで」を前に出すか、「複数鍵でハマる」を前に出すか
