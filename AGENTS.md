# AGENTS.md — obs-vrchat-stream セットアップ手順（AIエージェント向け）

このファイルは Claude Code・GitHub Copilot Agent・OpenAI Codex CLI 等の
AIエージェントが obs-vrchat-stream のセットアップを自動実行するための手順書です。

## 他エージェント環境での参照方法

- **Claude Code**: プロジェクトルートに置けば自動読み込み。サブディレクトリの場合は `@obs-vrchat-stream/AGENTS.md` で参照
- **GitHub Copilot**: `.github/copilot-instructions.md` にこのファイルの内容を埋め込むか参照を記述する
- **OpenAI Codex CLI**: `codex --instructions obs-vrchat-stream/AGENTS.md` で渡す

---

## プロジェクト概要

```
OBS (RTMP :1935) → MediaMTX (HLS :8888) → Tailscale Funnel (HTTPS) → VRChat
```

OBS の映像を VRChat ワールドの動画プレイヤーでリアルタイム再生する構成。
外部サーバー不要・費用無料（Tailscale 無料プランのみ使用）。

## ファイル構成

```
obs-vrchat-stream/
  setup.ps1           # MediaMTX 自動DL・yml設定生成（初回のみ実行）
  setup-once.cmd      # setup.ps1 のダブルクリック用ラッパー
  start-stream.ps1    # 配信開始（MediaMTX起動 + Tailscale Funnel有効化）
  start-server.cmd    # start-stream.ps1 のラッパー（pause付き）
  stop-stream.ps1     # 配信停止（Funnel停止 + MediaMTX停止）
  stop-server.cmd     # stop-stream.ps1 のラッパー
  bin/
    mediamtx.exe      # setup.ps1 が GitHub Releases からDL配置（setup前は存在しない）
    mediamtx.yml      # setup.ps1 が生成（setup前は存在しない）
  docs/
    user-guide.md     # ユーザー向け解説・手順書
```

---

## セットアップ手順

### Step 1: MediaMTX のセットアップ

`obs-vrchat-stream/` ディレクトリで実行する：

```powershell
cd obs-vrchat-stream
powershell -NoProfile -ExecutionPolicy Bypass -File setup.ps1
```

成功確認（両方 `True` になれば OK）：

```powershell
Test-Path bin\mediamtx.exe
Test-Path bin\mediamtx.yml
```

**setup.ps1 の動作内容:**
- GitHub API (`https://api.github.com/repos/bluenviron/mediamtx/releases/latest`) から最新版を取得
- `mediamtx_*_windows_amd64.zip` をダウンロードして `bin/mediamtx.exe` に配置
- 以下の内容で `bin/mediamtx.yml` を BOM なし UTF-8 で生成する:

```yaml
logLevel: info

rtmp: yes
rtmpAddress: :1935

hls: yes
hlsAddress: :8888
hlsVariant: mpegts        # lowLatency(fMP4)はVRChat AVProが音声を再生できないため必須
hlsSegmentCount: 3        # ウィンドウ3秒で遅延蓄積を防ぐ
hlsSegmentDuration: 1s

api: yes
apiAddress: 127.0.0.1:9997

paths:
  live:
    source: publisher
```

### Step 2: Tailscale の確認・インストール

インストール済み確認：

```powershell
$installed = ($null -ne (Get-Command tailscale -ErrorAction SilentlyContinue)) -or
             (Test-Path "C:\Program Files\Tailscale\tailscale.exe")
Write-Host "Tailscale installed: $installed"
```

`False` の場合はインストール：

```powershell
winget install --id Tailscale.Tailscale -e --accept-package-agreements --accept-source-agreements
```

### Step 3: Tailscale ログイン確認（⚠️ ユーザー操作が必要）

**この手順はブラウザ認証が必要なため、ユーザーに操作を依頼する。**

ログイン状態の確認（エージェントが実行）：

```powershell
& "C:\Program Files\Tailscale\tailscale.exe" status
```

`Logged out` / `NoState` / エラーが出た場合は、ユーザーに以下を依頼する：

> **ユーザーへ:** Tailscale にログインしてください。以下のいずれかを実行してブラウザでログインしてください。
> ```powershell
> "C:\Program Files\Tailscale\tailscale.exe" login
> ```
> または Tailscale アプリを開いてログインしてください。完了したら教えてください。

ログイン後の確認（IP アドレスが返れば準備完了）：

```powershell
& "C:\Program Files\Tailscale\tailscale.exe" ip -4
```

### Step 4: 配信サーバーの起動

```powershell
cd obs-vrchat-stream
powershell -NoProfile -ExecutionPolicy Bypass -File start-stream.ps1
```

成功時の出力例：

```
Server is ready
- Local URL:   http://127.0.0.1:8888/live/index.m3u8
- VRChat URL:  https://<device>.<tailnet>.ts.net/live/index.m3u8

OBS server: rtmp://localhost:1935/live
OBS stream key: stream
OBS keyframe interval: 1 second
```

**`VRChat URL` をユーザーに伝え、VRChat ワールドの動画プレイヤーへ入力するよう案内する。**

### Step 5: OBS 設定値をユーザーに案内する

エージェントは OBS を直接操作できないため、以下の設定値をユーザーへ伝える：

| 項目 | 値 |
|---|---|
| サービス | カスタム |
| サーバー | `rtmp://localhost:1935/live` |
| ストリームキー | `stream` |
| キーフレーム間隔 | `1` 秒（必須） |

---

## 停止・クリーンアップ

配信停止：

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File stop-stream.ps1
```

MediaMTX バイナリ削除（再セットアップが必要になる）：

```powershell
Remove-Item -Recurse -Force obs-vrchat-stream\bin
```

---

## 既知の問題・注意事項

### Tailscale Funnel 未有効 tailnet

`start-stream.ps1` が `Funnel is not enabled on your tailnet` を出す場合:
- スクリプトが自動でブラウザを開く（`https://login.tailscale.com/f/funnel?...`）
- ユーザーに有効化を依頼し、完了後に `start-stream.ps1` を再実行

### `tailscale funnel --bg --yes 8888` がハングする

Funnel 未有効 tailnet では有効化 URL 出力後にプロセスがハングする。
`start-stream.ps1` は ProcessStartInfo + タイムアウト 15 秒で URL を抽出するため、スクリプトをそのまま使えば自動対処される。

### `tailscale cert domains` が使えない環境

一部 tailnet で `tailscale cert domains` が "500 invalid domain" を返す。
`start-stream.ps1` は `tailscale funnel status` の出力から正規表現でドメインを取得するため問題ない。

### `Set-StrictMode -Version Latest` 環境での注意

`start-stream.ps1` は StrictMode 下で動作するよう設計されている。
`+=` やイベント登録 (`add_EventName`) は StrictMode のスコープ問題で動作しない場合があるため、
I/O 読み取りは `ReadToEndAsync()` をプロセス起動直後に開始し、`WaitForExit` 後に `GetResult()` する方式を採用している。

### mediamtx.yml の必須設定

- `hlsVariant: mpegts` **必須** — `lowLatency`（fMP4 Part）では VRChat の AVPro が音声を再生できない
- `hlsSegmentCount: 3` **推奨** — ウィンドウを 3 秒に保ち遅延蓄積を防ぐ
- `hlsSegmentDuration: 1s` — OBS のキーフレーム間隔と揃える

### Tailscale Funnel の停止コマンド

`tailscale funnel reset` は `--yes` フラグに非対応。`stop-stream.ps1` では `& $TailscaleExe funnel reset` で実行する。

### Tailscale サービスが停止している場合

`start-stream.ps1` はスクリプト冒頭で Tailscale サービスの起動を試みる。
`Get-Service -Name "Tailscale"` でサービス状態を確認し、停止中なら `Start-Service` する。
