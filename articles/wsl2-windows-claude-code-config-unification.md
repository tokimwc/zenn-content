---
title: "Windows + WSL2 で Claude Code の設定ファイルどうする問題 —— 「完全統一」の罠と現実解"
emoji: "🔗"
type: "tech"
topics: ["WSL2", "ClaudeCode", "Windows", "SSH", "環境構築"]
published: true
---

## はじめに

Windows + WSL2 で Claude Code を使っていると、ある日ふと気づきます。

**「`.claude` ディレクトリが2つある」**

```
C:\Users\<user>\.claude\       ← Windows ネイティブ側
/home/<user>/.claude/           ← WSL2 (Ubuntu) 側
```

認証情報、設定ファイル、メモリ、プロジェクト設定……全部別々です。Windows のデスクトップ版でログインしても、WSL2 側では未認証のまま。SSH 鍵も `.ssh` が2セット存在する。

「設定を一本化したい」と思って調べると、`CLAUDE_CONFIG_DIR` 環境変数で統一する方法が出てきます。しかし、これには落とし穴がありました。

本記事では、実際に環境を整理した経験をもとに、**現実的に壊れにくい構成**を紹介します。

## 環境

- Windows 11
- WSL2 (Ubuntu 24.04)
- Claude Code（VS Code 拡張 + デスクトップ版 + ターミナル版）
- SSH 接続先が複数（GitHub 複数アカウント含む）

## 問題の整理：何が分離しているのか

まず `.claude` ディレクトリの中身を確認しましょう。

| ファイル/ディレクトリ | 内容 | 環境依存 |
|---|---|---|
| `.credentials.json` | 認証情報（APIキー等） | **共有したい** |
| `settings.json` | 各種設定 | 環境依存あり（MCPサーバーのパス等） |
| `projects/` | プロジェクト別設定 | パスベースで管理される |
| `memory/` | AI のメモリ | プロジェクトパスに紐づく |
| `history.jsonl` | 対話履歴 | 環境固有 |

ここで重要なのは、**本当に共有したいのは `credentials` だけ**という点です。

`settings.json` には MCP サーバーのパスなど環境依存の値が入り得ます。`projects/` や `memory/` はプロジェクトのフルパスをキーにしているため、Windows（`C:\Users\...`）と WSL2（`/home/...`）では別エントリになります。

## よくある提案：`CLAUDE_CONFIG_DIR` で完全統一

検索すると、こんな提案が見つかります。

> WSL2 側の `~/.claude` を正とし、Windows 側の PowerShell プロファイルに以下を設定する：
> ```powershell
> $env:CLAUDE_CONFIG_DIR = "\\wsl$\Ubuntu\home\<user>\.claude"
> ```

一見スマートですが、**矛盾を抱えています**。

### 9P プロトコルの問題

WSL2 と Windows 間のファイルアクセスは、9P プロトコルを経由します。これは双方向で同じです。

- WSL2 から `/mnt/c/...` 経由で Windows ファイルにアクセス → 9P 経由（遅い）
- Windows から `\\wsl$\Ubuntu\...` 経由で WSL2 ファイルにアクセス → 9P 経由（遅い）

つまり「9P は非推奨」と言いながら、`CLAUDE_CONFIG_DIR` で `\\wsl$\...` を参照させるのは、**まさにその 9P を逆方向に使うだけ**です。

Windows のデスクトップ版やターミナル版 Claude Code が、毎回 WSL2 のファイルシステムを 9P 経由で読むことになり、動作が遅くなる可能性があります。

### `settings.json` の衝突

仮に 9P の速度を許容したとしても、`settings.json` を共有すると問題が起きます。

```json
// Windows 側の MCP サーバー設定
{
  "mcpServers": {
    "myTool": {
      "command": "C:\\tools\\my-mcp-server.exe"
    }
  }
}
```

このパスは WSL2 では動きません。環境ごとに異なる値を持つファイルを、無理に共有すべきではありません。

## 現実解：credentials だけシンボリックリンク

やるべきことはシンプルです。

### 方針

```
Windows側 (~\.claude\)          ← デスクトップ版でログインする場所
  └─ .credentials.json         ← 実体（正）

WSL2側 (~/.claude/)             ← AI 開発のメイン作業場
  └─ .credentials.json         ← シンボリックリンク → Windows側
  └─ settings.json             ← WSL2 専用設定（独立）

※ settings.json, memory/, projects/ は環境別に独立管理
```

### 手順

**1. WSL2 側の credentials をバックアップ**

```bash
cp ~/.claude/.credentials.json ~/.claude/.credentials.json.bak
```

**2. シンボリックリンクを作成**

```bash
ln -sf /mnt/c/Users/<user>/.claude/.credentials.json \
       ~/.claude/.credentials.json
```

**3. 確認**

```bash
ls -la ~/.claude/.credentials.json
# lrwxrwxrwx ... -> /mnt/c/Users/<user>/.claude/.credentials.json
```

これだけです。Windows のデスクトップ版でログイン（`claude login`）すれば、WSL2 側にも自動的に反映されます。

### シンボリックリンクの方向について

「どちらを実体にするか」は、**ログインする頻度が高い方を実体にする**のが自然です。

- デスクトップ版（Windows）でログインすることが多い → **Windows 側を実体**（上記の例）
- WSL2 でログインすることが多い → WSL2 側を実体にし、Windows から参照

ここで `/mnt/c/...` 経由の 9P アクセスが発生しますが、credentials の読み込みは起動時に一度だけなので、実用上問題になりません。`settings.json` のように常時参照されるファイルとは事情が違います。

## SSH 鍵の同期

SSH 鍵も Windows と WSL2 で分離していることがあります。

### やること：両方に同じ鍵をコピーする

クロス OS 参照（`IdentityFile /mnt/c/...` など）は避け、**単純にファイルをコピー**します。

```bash
# Windows → WSL2 にコピー
cp /mnt/c/Users/<user>/.ssh/id_ed25519_mykey     ~/.ssh/
cp /mnt/c/Users/<user>/.ssh/id_ed25519_mykey.pub  ~/.ssh/

# パーミッション設定（重要！WSL2 では必須）
chmod 600 ~/.ssh/id_ed25519_mykey
chmod 644 ~/.ssh/id_ed25519_mykey.pub
```

### SSH config も環境ごとに独立管理

Windows と WSL2 で同じ Host エントリを持つが、パスの書き方が異なります。

**Windows 側**（`C:\Users\<user>\.ssh\config`）:

```
Host my-server
    HostName 192.168.x.x
    User admin
    IdentityFile C:\Users\<user>\.ssh\id_ed25519
```

**WSL2 側**（`~/.ssh/config`）:

```
Host my-server
    HostName 192.168.x.x
    User admin
    IdentityFile ~/.ssh/id_ed25519
```

内容はほぼ同じですが、`IdentityFile` のパス形式が異なるため、共有ではなく**それぞれ管理**します。

## おまけ：複数 GitHub アカウントの SSH 使い分け

複数の GitHub アカウントを SSH で使い分ける場合、`~/.ssh/config` で **Host エイリアス**を使います。

### 構成例

2つの GitHub アカウントを持っている場合：

| アカウント | 用途 | SSH Host | 鍵 |
|---|---|---|---|
| `account-a` | 個人開発・趣味 | `github-personal` | `id_ed25519_personal` |
| `account-b` | 仕事・コミュニティ | `github.com`（デフォルト） | `id_ed25519_work` |

### SSH config

```
# 個人アカウント（エイリアス使用）
Host github-personal
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_personal
    IdentitiesOnly yes

# 仕事アカウント（デフォルト）
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_work
    IdentitiesOnly yes
```

### 使い分け方

```bash
# 個人アカウントのリポジトリ
git remote add origin git@github-personal:account-a/my-repo.git

# 仕事アカウントのリポジトリ（通常通り）
git remote add origin git@github.com:account-b/work-repo.git
```

`git config user.name` と `user.email` もリポジトリごとに設定すれば、コミットの著者情報も分離できます。

```bash
# リポジトリローカルで設定
git config user.name "account-a"
git config user.email "12345678+account-a@users.noreply.github.com"
```

## 各 Claude Code モードの使い分け

最後に、Windows + WSL2 環境での Claude Code の使い分けを整理します。

| 用途 | 推奨モード | 理由 |
|---|---|---|
| AI 開発・GPU 作業 | WSL2 ターミナル版（VS Code Remote WSL） | ファイルシステムがネイティブ Linux、`/mnt` 不要 |
| バックグラウンド並列タスク | Windows ターミナル版（複数端末） | 複数のターミナルを開いて並列実行 |
| 軽い作業・メモ整理 | Windows デスクトップ版 | UI がフレンドリー、bypass モードで効率的 |

ポイントは**作業プロジェクト自体を WSL2 のファイルシステム内に置く**ことです。WSL2 から `/mnt/c/...` 経由で Windows 側のプロジェクトを操作すると、9P のオーバーヘッドで遅くなります。

ドキュメント管理ツール（Obsidian 等）への書き込みなど、**Windows 側のファイルへのアクセスは最小限**にとどめるのがストレスの少ない構成です。

## まとめ

| やること | 方法 |
|---|---|
| credentials の共有 | シンボリックリンク1本（ログイン頻度が高い方を実体に） |
| settings / memory | 環境別に独立管理（共有しない） |
| SSH 鍵 | 両方に同じ鍵をコピー（パーミッション設定忘れずに） |
| SSH config | 環境ごとに独立管理（パス形式が異なるため） |
| 作業プロジェクト | WSL2 のネイティブファイルシステムに置く |

「完全統一」を目指すと、環境差異のデバッグに時間を取られて本末転倒になります。**共有すべきものだけ共有し、あとは独立管理**。これが一番壊れにくい構成でした。

---

*この記事は、Windows 11 + WSL2 (Ubuntu 24.04) + Claude Code の環境で実際に構築・運用している構成をもとに書いています。*