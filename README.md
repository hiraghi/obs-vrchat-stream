# obs-vrchat-stream

> OBS の映像を VRChat ワールドの動画プレイヤーにリアルタイム配信する — **外部サーバー不要**

![Platform](https://img.shields.io/badge/platform-Windows%2010%2F11-blue)
![License](https://img.shields.io/badge/license-MIT-green)

```
OBS ──RTMP──▶ MediaMTX ──HLS──▶ Tailscale Funnel ──HTTPS──▶ VRChat
```

- 🎮 ゲーム画面・VR 映像・Webカメラを VRChat の動画プレイヤーに映せる
- 👥 フレンドが同じワールドにいれば同時視聴できる
- ⏱️ 遅延約 3 秒（HLS 1 秒セグメント × 3 枚）
- 🆓 サーバー代・固定 IP・ポート開放 すべて不要

## Requirements

| ツール | 用途 |
|---|---|
| [OBS Studio](https://obsproject.com/ja) | 映像キャプチャ・RTMP 送信 |
| [Tailscale](https://tailscale.com/) アカウント | ローカルサーバーを HTTPS 公開（無料プラン可） |
| AI エージェント | セットアップの自動実行（後述） |

## Quick Start

### Step 1 — AI エージェントでセットアップ

`AGENTS.md` をコンテキストに渡して「セットアップして」と伝えるだけです。

```bash
# Claude Code
claude "obs-vrchat-stream をセットアップして"

# GitHub Copilot CLI
gh copilot suggest -t shell "obs-vrchat-stream をセットアップして。手順は AGENTS.md を参照"

# OpenAI Codex CLI
codex --instructions AGENTS.md "このプロジェクトをセットアップして"
```

エージェントが自動で実行します：

1. Tailscale のインストール確認（必要なら `winget` でインストール）
2. MediaMTX 最新版のダウンロード・配置
3. `bin/mediamtx.yml` の生成
4. 配信サーバー起動・**VRChat URL** の表示

> [!NOTE]
> Tailscale のログインのみブラウザ認証が必要です。エージェントから案内があったらブラウザでログインし、完了したとエージェントに伝えてください。

### Step 2 — OBS を設定する

OBS の「設定 → 配信」を以下に設定します（エージェントは OBS を直接操作できません）。

| 項目 | 値 |
|---|---|
| サービス | カスタム |
| サーバー | `rtmp://localhost:1935/live` |
| ストリームキー | `stream` |
| キーフレーム間隔 | **1 秒**（必須） |

> [!WARNING]
> キーフレーム間隔が 1 秒でないと映像が遅延し続けて正常に再生できません。

## Usage

セットアップ後は `start-server.cmd` / `stop-server.cmd` をダブルクリックするだけです。

```
Server is ready
- Local URL:   http://127.0.0.1:8888/live/index.m3u8
- VRChat URL:  https://xxxxxxxx.tail123456.ts.net/live/index.m3u8
```

1. `VRChat URL` をコピーして VRChat ワールドの動画プレイヤーに貼り付ける
2. OBS で「配信開始」をクリックする

> [!TIP]
> URL は `/live/index.m3u8` で終わる形式で入力してください。`/live/` だけでは再生できません。

停止するときは `stop-server.cmd` をダブルクリックし、OBS の「配信停止」も行ってください。

## How it works

| コンポーネント | 役割 | 費用 |
|---|---|---|
| **OBS Studio** | 映像を RTMP で送信 | 無料 |
| **MediaMTX** | RTMP → HLS 変換（自動 DL） | 無料 |
| **Tailscale Funnel** | ローカルサーバーを HTTPS で外部公開 | 無料プラン可 |
| **VRChat** | HLS URL を動画プレイヤーで再生 | 無料 |

Tailscale Funnel を使うことでルーターのポート開放やダイナミック DNS が不要になります。

## Troubleshooting

<details>
<summary><code>no current Tailscale IPs; state: NoState</code> が出る</summary>

Tailscale がログアウト状態です。

1. タスクトレイの Tailscale アイコンをクリックして接続状態にする
2. または `"C:\Program Files\Tailscale\tailscale.exe" up` を実行
3. `start-server.cmd` を再実行

</details>

<details>
<summary><code>Funnel is not enabled on your tailnet</code> が出る</summary>

ブラウザが自動で開きます（開かない場合は表示された URL を手動で開いてください）。Tailscale 管理画面で Funnel を **Enable** してから `start-server.cmd` を再実行してください。

Funnel の有効化は tailnet ごとに 1 回のみ必要です。

</details>

<details>
<summary>ローカルでは再生できるが VRChat で映らない</summary>

- URL が `https://...ts.net/live/index.m3u8` の形式になっているか確認
- VRChat ワールドのプレイヤーが HLS（AVPro）対応かどうか確認

</details>

<details>
<summary>映像は映るが音が出ない</summary>

エージェントに「obs-vrchat-stream を再セットアップして」と依頼して `bin/mediamtx.yml` を最新設定で上書きしてもらってください。

</details>

<details>
<summary>映像が遅延し続けて追いつかない</summary>

OBS のキーフレーム間隔が **1 秒** になっているか確認してください。

</details>

## License

MIT
