# youtuber

ローカルの動画を、自分の YouTube チャンネルに **非公開** でアップロードし、既存の **非公開プレイリスト** に追加するための Claude Code スキルと CLI。

- 公開範囲は **private 固定**。公開・限定公開にする手段は CLI にない
- 子ども向け設定は常に「いいえ」
- 自分の非公開プレイリスト以外には追加しない
- アップロード前に、チャンネル・ファイル・タイトル・プレイリストを示してユーザーの承認を取る
- 同じファイルを再実行しても二重にアップロードしない

| パス | 内容 |
|---|---|
| `skills/yt-uploader/` | アップロード用スキルと CLI（`scripts/yt_uploader.py`） |
| `skills/yt-uploader-setup/` | セットアップ用スキル |
| `.claude/skills/` | 上の 2 つへのリンク（Claude Code が読み込む） |
| `.claude/settings.json` | `upload` 実行時に必ず確認を出す設定 |
| `docs/idea.md` | 調査・設計メモ |

## セットアップ

必要なもの：[uv](https://docs.astral.sh/uv/)、Google アカウント（YouTube チャンネルあり）。

Claude Code で「**yt-uploader をセットアップして**」と頼むと、下の手順を順に案内します。手動で行う場合は次のとおりです。

### 1. Google Cloud の準備（最初の 1 回）

1. https://console.cloud.google.com/ でプロジェクトを新規作成する（名前は自由）。
2. 「API とサービス」→「ライブラリ」で **YouTube Data API v3** を有効にする。
3. 「Google Auth Platform」（または「OAuth 同意画面」）で次を設定する。
   - 対象：**外部（External）**
   - アプリ名・メール：自分のもの
   - テストユーザー：動画を上げるアカウント
4. 「クライアント」→ 種類 **デスクトップアプリ** で作成し、JSON をダウンロードする。

### 2. JSON を置く

```bash
mkdir -p ~/.config/yt-uploader
mv ~/Downloads/client_secret_*.json ~/.config/yt-uploader/client_secret.json
chmod 600 ~/.config/yt-uploader/client_secret.json
```

### 3. ログイン

リポジトリのルートで実行する（Claude Code 内なら先頭に `!` を付ける）。

```bash
uv run skills/yt-uploader/scripts/yt_uploader.py auth
```

ブラウザでアカウント（とチャンネル）を選び、「このアプリは確認されていません」→「詳細」→「移動」→「許可」。
`"ok": true` とチャンネル名が出れば完了。

### 4. 確認

```bash
uv run skills/yt-uploader/scripts/yt_uploader.py playlists
```

> 同意画面が「テスト中」のままだと、約 7 日でログインが切れることがあります。切れたら 3 をやり直してください。
> 同意画面で「アプリを公開」に切り替えると期限はなくなります（未確認アプリの警告は出続けます）。

## 使い方

Claude Code で「**この動画を YouTube に保存して** `~/Movies/xxx.mp4`」のように頼むと、プレイリスト候補とタイトル案を出し、承認後にアップロードします。

CLI を直接使う場合：

```bash
uv run skills/yt-uploader/scripts/yt_uploader.py whoami      # 認証中のチャンネル
uv run skills/yt-uploader/scripts/yt_uploader.py playlists   # 非公開プレイリスト一覧
uv run skills/yt-uploader/scripts/yt_uploader.py upload /path/video.mp4 \
  --channel UC... --title "タイトル" [--description "説明"] [--playlist PL...]
```

出力は JSON。失敗時は `"ok": false` と `error` コードを返し、終了コード 1。

## 保存されるもの

`~/.config/yt-uploader/`（ディレクトリ 700・ファイル 600、`YT_UPLOADER_HOME` で変更可）

- `client_secret.json`：OAuth クライアント
- `token.json`：ログイン情報
- `ledger.json`：アップロード済みファイル（SHA-256）と動画 ID の記録。再実行時の重複防止に使う

いずれもリポジトリには入りません。

## テスト

実 API には触れません。

```bash
uv run skills/yt-uploader/scripts/test_yt_uploader.py
```

## 注意

- YouTube の **非公開はバックアップではありません**。元ファイルは手元にも残してください。
- YouTube API の開発者ポリシーは、アップロード時に公開範囲をユーザーが選べることを求めています。このツールは自分専用の private 固定ツールで、その要件を満たしていません。他人に配布する場合は見直しが必要です。
- 同じ OS ユーザーで動く AI は `token.json` に技術的には到達できます。スキルで禁止していますが、完全な隔離ではありません。
- 権限確認をすべて許可するモード（bypass）では、`upload` 前の確認が出ません。ユーザー設定に `Bash(uv run:*)` のような広い許可ルールを入れると、書き方によっては確認が抜けることがあります。

## 開発環境（任意）

Nix + direnv + mise のテンプレート構成です。

```bash
curl --proto '=https' --tlsv1.2 -sSf -L https://install.determinate.systems/nix | sh -s -- install
```

> **Determinate Systems 版を推奨。** `nix-command` と `flakes` が最初から有効なため、追加設定不要。
