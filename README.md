# 開発コンテナで Claude Code を使う手順

## 概要

このリポジトリには、Claude Code をセキュアな開発コンテナ内で実行するための設定が含まれています。
コンテナはネットワークを制限したサンドボックス環境として動作し、許可されたドメインのみとの通信を許可します。

## ファイル構成

```
.devcontainer/
├── devcontainer.json   # コンテナの設定（VS Code Dev Containers 用）
├── Dockerfile          # Ubuntu ベースのコンテナ定義
└── init-firewall.sh    # 起動時に実行されるファイアウォール設定スクリプト
```

## 前提条件

- Docker Desktop がインストール済みであること
- VS Code と [Dev Containers 拡張機能](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) がインストール済みであること
- Claude Code の VS Code 拡張機能（`anthropic.claude-code`）— コンテナ起動時に自動インストールされます

## 起動手順

1. VS Code でこのリポジトリを開く
2. コマンドパレット（`Cmd+Shift+P`）から **"Dev Containers: Reopen in Container"** を選択する
3. コンテナのビルドとファイアウォールの初期化が完了するまで待つ（初回は数分かかります）
4. ターミナルで `claude` コマンドを実行する

## コンテナの設定詳細

### Dockerfile

| 項目 | 内容 |
|------|------|
| ベースイメージ | `ubuntu:26.04` |
| デフォルトユーザー | `ubuntu` |
| 作業ディレクトリ | `/workspace` |
| インストール済みツール | git, zsh, fzf, gh, curl, jq, iptables, ipset など |
| Claude Code | 起動スクリプト（`https://claude.ai/install.sh`）で自動インストール |

### devcontainer.json の主要設定

| 設定 | 値 | 説明 |
|------|----|----|
| `remoteUser` | `ubuntu` | コンテナ内の実行ユーザー |
| `runArgs` | `NET_ADMIN`, `NET_RAW` | iptables 操作に必要なケーパビリティ |
| `postStartCommand` | `sudo /usr/local/bin/init-firewall.sh` | 起動後にファイアウォールを初期化 |
| `CLAUDE_CONFIG_DIR` | `/home/ubuntu/.claude` | Claude の設定ディレクトリ |

### マウントされるボリューム

| ボリューム | コンテナ内パス | 用途 |
|-----------|---------------|------|
| `claude-code-bashhistory-*` | `/commandhistory` | Bash 履歴の永続化 |
| `claude-code-config-*` | `/home/ubuntu/.claude` | Claude の設定・認証情報の永続化 |
| ローカルのワークスペース | `/workspace` | プロジェクトファイル（バインドマウント） |

設定とコマンド履歴はコンテナを再作成しても保持されます。

## ファイアウォールの仕組み

コンテナ起動後、`init-firewall.sh` が自動実行され、iptables と ipset によるネットワーク制限が適用されます。

### 許可されるアウトバウンド通信

| 対象 | 用途 |
|------|------|
| GitHub（`.web`, `.api`, `.git` の全 IP レンジ） | リポジトリ操作・API アクセス |
| `api.anthropic.com` | Claude API |
| `sentry.io` | エラー監視 |
| `statsig.com` | 機能フラグ |
| `marketplace.visualstudio.com` | VS Code 拡張機能 |
| `vscode.blob.core.windows.net` | VS Code アセット |
| `update.code.visualstudio.com` | VS Code アップデート |
| ホストネットワーク（`/24`） | ローカル開発環境との通信 |
| DNS（UDP 53）・SSH（TCP 22）・ループバック | 基本通信 |

### 遮断されるアウトバウンド通信

上記以外のすべての通信は `REJECT` されます。起動スクリプトは以下を確認して終了します。

- `https://example.com` に**接続できない**（制限が有効）
- `https://api.github.com` に**接続できる**（許可が有効）

### ファイアウォール設定の流れ

```
1. 既存のルールをすべてフラッシュ
2. Docker 内部 DNS ルールを復元
3. DNS・SSH・ループバックを許可
4. GitHub の IP レンジを取得して ipset に追加
5. 許可ドメインを DNS 解決して ipset に追加
6. ホストネットワークを許可
7. デフォルトポリシーを DROP に設定
8. ipset に一致するアウトバウンドのみ許可
9. 動作検証（接続テスト）
```

## トラブルシューティング

### ファイアウォールの初期化に失敗する

コンテナ起動時に GitHub の IP レンジ取得で失敗する場合は、SSL 証明書ストアの問題の可能性があります。ターミナルで以下を実行してください。

```bash
sudo /usr/local/bin/init-firewall.sh
```

### Claude Code の認証が必要な場合

コンテナ内のターミナルで以下を実行します。

```bash
claude
```

初回起動時は認証フローが開始されます。認証情報は `/home/ubuntu/.claude` ボリュームに永続化されるため、コンテナを再作成しても再認証は不要です。

### タイムゾーンを変更したい場合

`devcontainer.json` の `TZ` ビルド引数はホスト環境変数 `TZ` から取得します。ホストで `TZ` が未設定の場合は `America/Los_Angeles` が使われます。変更するにはコンテナのリビルドが必要です。

```json
"args": {
  "TZ": "Asia/Tokyo"
}
```
