# Claude Code 権限システム仕様

更新日: 2026-03-21


## 概要

Claude Code はツール実行時にユーザーへ権限確認を行う。\
`settings.json` の設定により、確認頻度を制御できる。


## 権限モード

| モード | 動作 | 用途 |
| --- | --- | --- |
| `default` | 各ツール初回使用時に確認 | 通常開発 |
| `acceptEdits` | ファイル編集を自動承認、他は確認 | 信頼済みの反復作業 |
| `plan` | 分析のみ、ファイル変更・コマンド実行不可 | コードレビュー、変更提案 |
| `bypassPermissions` | 保護ディレクトリ以外の確認をスキップ | コンテナ・隔離 VM 限定 |


## 設定ファイルの優先順位

上位が優先される。

1. マネージド設定（管理者制御、上書き不可）
2. コマンドライン引数（セッション限定）
3. `.claude/settings.local.json`（ローカルプロジェクト、コミット対象外）
4. `.claude/settings.json`（プロジェクト共有、コミット対象）
5. `~/.claude/settings.json`（ユーザーグローバル）


## 権限ルールの構造

`allow`、`ask`、`deny` の3種類のルールを定義できる。

```json
{
  "permissions": {
    "allow": [
      "Read",
      "Bash(npm run *)",
      "Bash(git commit *)"
    ],
    "ask": [
      "Bash(git push *)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(sudo *)"
    ]
  }
}
```


### ルール評価順序

1. **deny** — 最初にマッチしたら即ブロック（allow より優先）
2. **ask** — 確認ダイアログを表示
3. **allow** — 自動承認


## ツール別の設定方法


### Bash コマンド

ワイルドカード `*` でパターン指定できる。

```json
"Bash(npm run *)"      // npm run build, npm run test 等にマッチ
"Bash(git add *)"      // git add . 等にマッチ
"Bash(git * main)"     // git checkout main, git merge main 等にマッチ
```

> **単語境界に注意**: `Bash(ls *)` は `ls -la` にマッチするが `lsof` にはマッチしない。\
> スペースなしの `Bash(ls*)` は両方にマッチする。

> **複合コマンド**: `git status && npm test` を承認すると、各サブコマンドに対して個別にルールが保存される（最大5個）。


### Read / Edit（パスパターン）

gitignore 仕様に準拠した4種類のパス指定が使える。

```json
"Read"                      // 全ファイル読み取り許可
"Edit(/src/**/*.ts)"        // プロジェクトルート相対
"Read(~/Documents/*.pdf)"   // ホームディレクトリ相対
"Read(//etc/hosts)"         // ファイルシステム絶対パス
```


### WebFetch

ドメイン単位で制御する。

```json
"WebFetch(domain:github.com)"
"WebFetch(domain:api.example.com)"
```


### MCP サーバーツール

```json
"mcp__puppeteer"                       // サーバーの全ツール
"mcp__puppeteer__puppeteer_navigate"   // 特定ツール
"mcp__slack__*"                        // ワイルドカード
```


## 確認を必ず維持すべき操作

以下は安全上、`deny` または `ask` に設定し、`allow` にすべきではない。

- `Bash(git push *)` — リモートへの変更反映
- `Bash(git push --force *)` — 履歴の上書き
- `Bash(rm -rf *)` — 不可逆な削除
- `Bash(sudo *)` — 特権操作
- `Edit(//.git/**)` — Git メタデータの変更
- `Edit(//.env)` — シークレットファイルの変更

> **Read/Edit の deny は Bash をブロックしない。**\
> `Read(//.env)` を deny しても `Bash(cat .env)` は防げない。\
> 機密ファイルは `Bash(cat .env)` も deny に追加するか、OS レベルのサンドボックスで保護する。


## 安全に自動承認できる操作

以下は `allow` に設定しても安全性への影響が小さい。

```json
{
  "permissions": {
    "allow": [
      "Read",
      "Bash(npm run build)",
      "Bash(npm run test)",
      "Bash(npm run dev)",
      "Bash(npm run lint)",
      "Bash(npm run lint:fix)",
      "Bash(npm install)",
      "Bash(npx *)",
      "Bash(git add *)",
      "Bash(git commit *)",
      "Bash(git checkout *)",
      "Bash(git status)",
      "Bash(git diff *)",
      "Bash(git log *)",
      "Bash(git branch *)",
      "Bash(node --version)",
      "Bash(npm --version)"
    ]
  }
}
```


## 確認回数を減らすための方法

1. **`settings.json` に頻出コマンドを登録する。**\
プロジェクト固有のビルド・テストコマンドを `allow` に追加する。

2. **セッション中の「Yes, don't ask again」を活用する。**\
承認時にこのオプションを選ぶと、セッション中は同種の操作を自動承認する。

3. **`acceptEdits` モードを使う。**\
方針を確認した後、ファイル編集の確認をスキップする。

4. **プロジェクトレベルの `.claude/settings.json` を作成する。**\
チームで合意した安全なデフォルトを共有する。

5. **`deny` + `WebFetch` を組み合わせる。**\
`Bash(curl *)` を deny し、`WebFetch(domain:...)` でドメイン制限する方が、Bash パターンで URL を制御するより安全。


## Bash パターンの制限事項

Bash のパターンマッチングには構造上の限界がある。

- `curl http://github.com/*` は `curl -X GET http://github.com/...`（オプションが先頭）にマッチしない
- `https://` プロトコルやリダイレクトを考慮しない
- 変数展開（`URL=http://... && curl $URL`）を検出できない

ネットワークアクセスの制御は Bash パターンではなく、`WebFetch` のドメイン制限を使うこと。


## --dangerously-skip-permissions フラグ

```bash
claude "your task" --dangerously-skip-permissions
```

全ての権限確認をスキップする。\
`bypassPermissions` モードと同等。

**使用してよい場面:**

- CI/CD コンテナ
- Docker 開発環境
- 使い捨て VM

**使用してはいけない場面:**

- 重要なファイルがある開発マシン
- 共有システム
- 本番サーバー


## 保護ディレクトリ

`bypassPermissions` モードでも、以下のディレクトリへの書き込みは確認が必要。

- `.git/**`
- `.claude/**`（ただし `.claude/commands`、`.claude/agents`、`.claude/skills` は例外）
- `.vscode/**`
- `.idea/**`


## /permissions コマンド

```
/permissions
```

現在有効な全ルール（allow / ask / deny）を一覧表示する。\
どの設定ファイルから読み込まれたかも確認できる。


## CLAUDE.md と settings.json の役割の違い

| 項目 | CLAUDE.md | settings.json |
| --- | --- | --- |
| 役割 | プロンプトへの指示（振る舞いのガイド） | ツール実行の権限制御 |
| 権限制御 | 不可 | 可能 |
| 影響範囲 | Claude の応答・判断に影響 | ツール呼び出し時の確認ダイアログに影響 |

CLAUDE.md に「このフォルダは編集してよい」と書いても、ツール実行時の権限確認はスキップされない。\
権限の事前承認は必ず `settings.json` の `permissions.allow` で行う。


## 本プロジェクトの設定例

`~/.claude/settings.json` に以下を設定し、開発時の確認回数を最小化している。

### allow（自動承認）

**ファイル操作:**

```json
"Read",
"Edit(/anytime-markdown/**)",
"Edit(/anytime-markdown-docs/**)",
"Edit(/home/node/.claude/**)",
"Write(/anytime-markdown/**)",
"Write(/anytime-markdown-docs/**)",
"Write(/home/node/.claude/**)"
```

> `Read` はパス制限なしで全ファイルを許可。\
> `Edit` / `Write` はプロジェクトディレクトリと Claude Code 設定ディレクトリに限定。

**Bash コマンド:**

```json
"Bash(npm *)",
"Bash(npx *)",
"Bash(git status *)",
"Bash(git diff *)",
"Bash(git log *)",
"Bash(git add *)",
"Bash(git commit *)",
"Bash(git checkout *)",
"Bash(git merge *)",
"Bash(git pull *)",
"Bash(git branch *)",
"Bash(docker *)",
"Bash(bash *)"
```

### ask（都度確認）

```json
"Bash(git push *)",
"Bash(git push)"
```

> リモートへの変更反映は常に確認を維持する。

### deny（常にブロック）

```json
"Bash(rm -rf *)",
"Bash(sudo *)"
```


### 旧書式からの移行に関する注意

Bash パターンの書式が変更されている。

| 旧書式 | 新書式 |
| --- | --- |
| `Bash(git status:*)` | `Bash(git status *)` |
| `Bash(npm:*)` | `Bash(npm *)` |

`:*` 形式は旧バージョンの書式。\
現行バージョンでは `Bash(コマンド *)` のスペース区切りが正しいパターン。


## 補足: セッション中の権限モード切替

`Shift+Tab` でセッション中に権限モードを即座に切り替えられる。\
方針確認後に `acceptEdits` へ切り替え、実装フェーズでのファイル編集確認をスキップする使い方が実用的。


## ベストプラクティス（2026年時点）


### 権限モードの使い分け

| フェーズ | 推奨モード | 理由 |
| --- | --- | --- |
| 初回・不慣れなコード | `default` | 全操作を確認し、意図しない変更を防ぐ |
| 方針確定後の実装 | `acceptEdits` | ファイル編集の確認をスキップし、Bash は維持 |
| コードレビュー・分析 | `plan` | 読み取りのみ、変更不可 |
| CI/CD・隔離環境 | `bypassPermissions` | サンドボックス環境限定 |

> `acceptEdits` に切り替える前に、必ず Git コミットしておくこと。\
> 意図しない変更があっても `git checkout` で復元できる。


### deny リスト設計（ブロックリストファースト）

2026年の実践では「deny を先に定義し、allow は必要最小限に追加する」アプローチが推奨されている。\
全ユーザーが `default` モードであっても deny リストを設定すべき。

**deny に含めるべき操作:**

- `Bash(rm -rf *)` — 不可逆な削除
- `Bash(sudo *)` — 特権操作
- `Bash(git push --force *)` — 履歴の上書き
- `Bash(chmod 777 *)` — 過度な権限付与
- `Bash(curl *)` / `Bash(wget *)` — 任意の外部通信（WebFetch のドメイン制限を代用）

> `Bash(DROP TABLE *)` など、データベース操作を含むプロジェクトでは SQL 破壊コマンドも deny に含める。


### PreToolUse フック（高度な制御）

`settings.json` のパターンマッチングだけでは対応できないケースには、PreToolUse フックを使う。\
ツール実行前にシェルスクリプトで検査し、終了コード `0` で許可、`2` でブロックできる。

```bash
# ~/.claude/hooks/pre-tool-use.sh の例
# 危険なパターンをブロック
if echo "$INPUT" | grep -q "rm -rf /"; then
  echo "Blocked: dangerous rm command" >&2
  exit 2
fi
exit 0
```

> パターンマッチングの限界（変数展開やオプション順序の違い等）を補完する手段として有効。


### MCP サーバーの権限管理

- インストール前にサーバーのソースコードと公式ドキュメントを確認する
- 信頼されていないコンテンツを取得する MCP サーバーはプロンプトインジェクションのリスクがある
- `enableAllProjectMcpServers: true` は利便性が高いが、未検証のサーバーも有効化されるため注意が必要
- 明示的に承認したサーバーのみ有効化するのが安全


### サンドボックスとの併用

権限システムとサンドボックスは二重防御として機能する。

| 層 | 役割 |
| --- | --- |
| 権限システム | 「このツールを実行すべきか？」を判断 |
| サンドボックス | 「実行時に何にアクセスできるか？」を制限 |

> `Read(//.env)` を deny しても `Bash(cat .env)` は防げない。\
> OS レベルのサンドボックス（macOS: Seatbelt、Linux: bubblewrap）で補完する。

**避けるべきサンドボックス設定:**

- `allowUnixSockets` — Docker ソケット経由でサンドボックス回避のリスク
- `$PATH` 配下や `.bashrc` 等への書き込み許可 — 権限昇格のリスク


### エンタープライズ・チーム環境

チーム全体で統一ポリシーを適用する場合は `managed-settings.json` を使う。\
個別ユーザーによる上書きができない強制設定。

| OS | パス |
| --- | --- |
| Linux / WSL | `/etc/claude-code/managed-settings.json` |
| Windows | `C:\ProgramData\ClaudeCode\managed-settings.json` |

プロジェクト固有の設定は `.claude/settings.json`（コミット対象）で共有し、個人差分は `.claude/settings.local.json`（`.gitignore` に追加）で管理する。


### 実装時チェックリスト

- deny リストに破壊的コマンドを登録したか
- `acceptEdits` 切替前に Git コミットしたか
- MCP サーバーはソースを確認して承認したか
- `.claude/settings.local.json` を `.gitignore` に追加したか
- `bypassPermissions` はサンドボックス環境でのみ使用しているか


## 参考資料

| 記事 | 内容 |
| --- | --- |
| [Claude Code settings.json: Complete config guide](https://www.eesel.ai/blog/settings-json-claude-code) | ワイルドカードパターンの具体例が豊富な設定ガイド |
| [A complete guide to Claude Code permissions](https://www.eesel.ai/blog/claude-code-permissions) | 権限ルール評価順序・設定階層・セキュリティ考慮事項 |
| [YOLO Mode: Permission configuration](https://dev.to/rajeshroyal/yolo-mode-when-youre-tired-of-claude-asking-permission-for-everything-2daf) | `Shift+Tab` での権限モード即時切替、`acceptEdits` の実践的使い分け |
| [Claude Code Auto Approve guide](https://smartscope.blog/en/generative-ai/claude/claude-code-auto-permission-guide/) | `acceptEdits` モードの日常開発での活用シーン |
| [Claude Code Security Best Practices](https://www.backslash.security/blog/claude-code-security-best-practices) | allow/deny リスト設計の推奨パターン、MCP サーバーセキュリティ |
| [Better Claude Code permissions](https://blog.korny.info/2025/10/10/better-claude-code-permissions) | 「ルーチンで低リスクな操作のみ allow」の設計思想 |
| [dangerously-skip-permissions: Safe Usage Guide](https://www.ksred.com/claude-code-dangerously-skip-permissions-when-to-use-it-and-when-you-absolutely-shouldnt/) | bypassPermissions の安全な使用シーンと注意点 |
| [How I use Claude Code (+ my best tips)](https://www.builder.io/blog/claude-code) | 開発チームの daily workflow での実践的な権限設定例 |
| [Best Practices for Claude Code](https://code.claude.com/docs/en/best-practices) | Anthropic 公式のベストプラクティス |
| [Hooks reference](https://code.claude.com/docs/en/hooks) | PreToolUse フックの公式リファレンス |
| [Sandboxing](https://code.claude.com/docs/en/sandboxing) | サンドボックス設定の公式ドキュメント |
| [Enterprise configuration](https://support.claude.com/en/articles/12622667-enterprise-configuration) | managed-settings.json によるチーム統一管理 |
| [Secure Your Claude Skills with Custom PreToolUse Hooks](https://egghead.io/secure-your-claude-skills-with-custom-pre-tool-use-hooks~dhqko) | PreToolUse フックの実装チュートリアル |
| [trailofbits/claude-code-config](https://github.com/trailofbits/claude-code-config) | Trail of Bits による厳密なセキュリティ設定テンプレート |
