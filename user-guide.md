---
title: "OBSの映像をVRChatワールドで流す — MediaMTX × Tailscale Funnel 無料構築ガイド"
emoji: "📡"
type: "tech"
topics: ["vrchat", "obs", "tailscale", "hls", "mediamtx"]
published: false
---

OBS で録画・配信している映像（ゲーム画面・VR 映像・Webカメラなど）を、VRChat ワールドの動画プレイヤーでリアルタイム再生する方法を解説します。

外部サーバーなし・費用 **無料**（Tailscale 無料プランのみ使用）。セットアップは AIエージェントに任せれば、ダウンロード・設定生成・起動確認まで自動で完結します。

## できること

- 自分の PC 画面・ゲーム映像・カメラ映像を VRChat の動画プレイヤーに映す
- フレンドが同じワールドにいれば同時に見られる
- 遅延は約 3 秒（HLS セグメント 3 枚 × 1 秒）
- サーバー代・固定 IP 不要、ルーターのポート開放も不要

## 仕組みの概要

```
OBS ──RTMP──▶ MediaMTX ──HLS──▶ Tailscale Funnel ──HTTPS──▶ VRChat
(映像送信)    (形式変換)          (インターネット公開)          (動画プレイヤー)
```

| コンポーネント | 役割 | 費用 |
|---|---|---|
| **OBS Studio** | 映像をキャプチャして RTMP で送信 | 無料 |
| **MediaMTX** | RTMP → HLS 変換するローカルサーバー | 無料（自動DL） |
| **Tailscale Funnel** | ローカルサーバーを HTTPS で外部公開 | 無料プランで可 |
| **VRChat** | HLS URL を動画プレイヤーに入力して再生 | 無料 |

:::message
**なぜ Tailscale Funnel？**
自宅 PC で動かすサーバーは通常インターネットから見えません。Tailscale Funnel は Tailscale のインフラを経由して HTTPS で外部公開する機能で、ルーターのポート開放やダイナミック DNS が不要です。
:::

## 必要なもの

- Windows 10 / 11（64 ビット）
- [OBS Studio](https://obsproject.com/ja)（インストール済み）
- [Tailscale アカウント](https://tailscale.com/)（無料プランで可）
- このプロジェクトのフォルダ（`obs-vrchat-stream/`）
- Claude Code・GitHub Copilot Agent・OpenAI Codex CLI のいずれか

## セットアップ（初回のみ）

### 1. AIエージェントで自動セットアップ

`AGENTS.md` をコンテキストとして渡し、「このプロジェクトをセットアップして」と伝えるだけです。

```bash
# Claude Code の場合（プロジェクトルートで自動読み込み）
claude "obs-vrchat-stream をセットアップして"

# OpenAI Codex CLI の場合
codex --instructions obs-vrchat-stream/AGENTS.md "このプロジェクトをセットアップして"
```

エージェントが以下をすべて自動で行います：

- Tailscale のインストール確認（未インストールなら `winget` でインストール）
- MediaMTX の最新版を GitHub からダウンロード・配置
- 設定ファイル（`bin/mediamtx.yml`）の生成
- 配信サーバーの起動と **VRChat URL** の表示

:::message
**Tailscale のログインだけはブラウザ認証が必要です。** エージェントから案内があったら、ブラウザで Tailscale アカウントにログインして、完了したとエージェントに伝えてください。
:::

### 2. OBS の設定

エージェントからセットアップ完了の報告を受けたら、OBS を設定します（OBS の操作はエージェントではできません）。

「設定 → 配信」を以下のように設定します。

| 項目 | 値 |
|---|---|
| サービス | **カスタム...** |
| サーバー | `rtmp://localhost:1935/live` |
| ストリームキー | `stream` |

「出力 → エンコーダ設定」または「設定 → 出力」でキーフレーム間隔を設定します。

| 項目 | 値 |
|---|---|
| キーフレーム間隔 | **1 秒** |

:::message alert
キーフレーム間隔は必ず **1 秒** に設定してください。これを守らないと映像が遅延し続けて正常に再生できなくなります。
:::

## 配信の開始

エージェントが `start-server.cmd` を実行して VRChat URL を表示します。2 回目以降は **`start-server.cmd`** をダブルクリックすれば起動できます。

```
Server is ready
- Local URL:   http://127.0.0.1:8888/live/index.m3u8
- VRChat URL:  https://xxxxxxxx.tail123456.ts.net/live/index.m3u8
```

1. OBS で **「配信開始」** をクリックします
2. `VRChat URL` をコピーして VRChat ワールドの動画プレイヤーに入力します

:::message
URL は **`/live/index.m3u8`** で終わる形式で入力してください。
`/live/` だけでは VRChat で再生できません。
:::

## 配信の停止

**`stop-server.cmd`** をダブルクリックします。

- Tailscale Funnel が停止します（外部からのアクセスが遮断される）
- MediaMTX が停止します

OBS の「配信停止」もあわせて行ってください。

## 初回のみ: Tailscale Funnel の有効化

`start-server.cmd` の実行時に以下のメッセージが出ることがあります：

```
Funnel is not enabled on your tailnet.
```

この場合、ブラウザが自動で開きます（または表示された `https://login.tailscale.com/f/funnel?...` を開いてください）。Tailscale 管理画面で Funnel を **Enable** してから `start-server.cmd` を再実行してください。

:::message
Funnel の有効化は tailnet ごとに **1 回だけ** 必要です。次回以降は自動で処理されます。
:::

## トラブルシューティング

### `no current Tailscale IPs; state: NoState` が出る

Tailscale がログアウト状態です。

1. タスクトレイの Tailscale アイコンをクリックして接続状態にする
2. または PowerShell で `"C:\Program Files\Tailscale\tailscale.exe" up` を実行
3. `start-server.cmd` を再実行

### `Failed to enable Funnel` が出る

1. Tailscale アプリを開いてログイン状態を確認する
2. `"C:\Program Files\Tailscale\tailscale.exe" up` を実行する
3. `start-server.cmd` を再実行する

### ローカルでは再生できるが VRChat で映らない

- URL が `https://...ts.net/live/index.m3u8` の形式になっているか確認
- VRChat ワールドのプレイヤーが HLS（AVPro）対応かどうか確認

### 映像は映るが音が出ない

エージェントに「obs-vrchat-stream を再セットアップして」と依頼して、`bin/mediamtx.yml` を最新の設定で上書きしてもらってください。

### 映像が遅延し続けて追いつかない

OBS のキーフレーム間隔が **1 秒** になっているか確認してください。

## まとめ

| ファイル | 用途 |
|---|---|
| `start-server.cmd` | 配信開始（毎回） |
| `stop-server.cmd` | 配信停止（毎回） |

配信中は VRChat から退室しても URL は有効なままです。URL を共有すれば複数人が同時視聴できます。
