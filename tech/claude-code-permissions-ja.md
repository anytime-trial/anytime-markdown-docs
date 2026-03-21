# Claude Code の権限システムを理解する — 確認ダイアログを減らしつつ安全性を保つ方法

更新日: 2026-03-21
著者: Claude Code v2.1.80 (claude-opus-4-6)
編集・監修: Kiyotaka Ueda
利用スキル: tech-article (2026-03-11)、markdown-output (2026-03-10)

> **AI 生成コンテンツに関する注意事項**
>
> この記事は AI（Claude Code）によって生成されたものである。内容の正確性には注意を払っているが、以下の点に留意されたい。
>
> - 記載内容は執筆時点の情報に基づく。ソフトウェアのバージョンアップにより仕様が変更される可能性がある
> - 最新の正確な情報は公式ドキュメントを参照すること
> - 記事中のイラストは AI が生成したものであり、概念の理解を助ける目的で挿入している


## はじめに

Claude Code を使い始めると、ファイル編集やコマンド実行のたびに権限確認のダイアログが表示される。\
安全性を担保する仕組みだが、頻繁に表示されると開発のリズムが崩れる。

この記事では、Claude Code の権限システムの全体像を解説し、「安全性を損なわずに確認回数を最小化する」ための具体的な設定方法を示す。\
権限モードの使い分け、`settings.json` の構成、そして 2026 年時点のベストプラクティスまでをカバーする。


## 前提知識

- Claude Code の基本操作（ターミナルでの対話的利用）を理解していること
- JSON ファイルの編集ができること
- Git の基本操作を知っていること


## 権限システムの全体像

![権限システムの概念図](images/permissions-overview.png)

Claude Code の権限システムは「このツールを実行してよいか」を判断する仕組みである。\
判断のロジックは、3種類のルール（allow / ask / deny）と、それらを評価する優先順位で構成されている。

つまり、権限システムを制御するとは「どのルールを、どの設定ファイルに、どう書くか」を決めることである。


### ルール評価の順序

ルールは以下の順序で評価され、最初にマッチしたルールが適用される。

1. **deny** — 即座にブロック。allow より常に優先される
2. **ask** — 確認ダイアログを表示する
3. **allow** — 自動承認。確認なしで実行される

> **注意**: deny ルールは絶対的である。allow に同じパターンを書いても deny が優先される。


### 設定ファイルの階層

設定ファイルは5段階の階層を持ち、上位が下位を上書きする。

| 優先度 | ファイル | 用途 |
| --- | --- | --- |
| 1（最高） | マネージド設定 | 管理者が強制するポリシー |
| 2 | コマンドライン引数 | セッション限定の一時設定 |
| 3 | `.claude/settings.local.json` | 個人のローカル設定（コミットしない） |
| 4 | `.claude/settings.json` | チーム共有の設定（コミットする） |
| 5（最低） | `~/.claude/settings.json` | ユーザーのグローバル設定 |

具体的には、チームで「`git push` は確認必須」というルールを `.claude/settings.json` に書いておけば、全メンバーに適用される。\
個人の好みは `.claude/settings.local.json` で上書きできるが、マネージド設定だけは上書きできない。


## 権限モードを使い分ける

Claude Code には4つの権限モードがあり、セッション中に `Shift+Tab` で即座に切り替えられる。

| モード | ファイル編集 | Bash 実行 | 適した場面 |
| --- | --- | --- | --- |
| `default` | 確認あり | 確認あり | 初回利用、不慣れなコード |
| `acceptEdits` | 自動承認 | 確認あり | 方針確定後の実装フェーズ |
| `plan` | 不可 | 不可 | コードレビュー、分析 |
| `bypassPermissions` | 自動承認 | 自動承認 | コンテナ・隔離 VM 限定 |

実務で最も使いやすいのは `acceptEdits` である。\
ファイル編集の確認をスキップしつつ、Bash コマンドの安全チェックは維持できる。

> `acceptEdits` に切り替える前に、必ず Git コミットしておくこと。\
> 意図しない変更があっても `git checkout` で復元できる。


## settings.json で確認回数を減らす

![設定ファイルの概念図](images/settings-structure.png)

権限モードの切り替えはセッション単位だが、`settings.json` に書いたルールは永続的に適用される。\
頻繁に使うコマンドを `allow` に登録しておけば、セッションをまたいで確認が不要になる。


### 基本の構造

```json
{
  "permissions": {
    "allow": [
      "Read",
      "Bash(npm run build)",
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

`allow` に書いたパターンにマッチする操作は確認なしで実行される。\
`deny` に書いたパターンは、たとえ `allow` にも書いてあってもブロックされる。


### ツール別のパターン記法

#### Bash コマンド

ワイルドカード `*` でパターンを指定する。

```json
"Bash(npm run *)"       // npm run build, npm run test 等にマッチ
"Bash(git * main)"      // git checkout main, git merge main 等にマッチ
```

> **単語境界に注意**: `Bash(ls *)` は `ls -la` にマッチするが `lsof` にはマッチしない。\
> スペースなしの `Bash(ls*)` は両方にマッチする。

#### Read / Edit / Write（パス制限）

gitignore と同じパス記法で、対象ディレクトリを制限できる。

```json
"Read"                           // 全ファイル（パス制限なし）
"Edit(/src/**/*.ts)"             // プロジェクトルート相対
"Write(/home/user/project/**)"   // 絶対パス（// で始める）
```

#### WebFetch / MCP

```json
"WebFetch(domain:github.com)"                    // ドメイン単位
"mcp__plugin_serena_serena__find_symbol"          // MCP ツール単位
```


### 実用的な設定例

以下は、日常の開発で確認回数を最小化しつつ安全性を維持する設定である。

```json
{
  "permissions": {
    "allow": [
      "Read",
      "Edit(/your-project/**)",
      "Write(/your-project/**)",
      "Bash(npm *)",
      "Bash(npx *)",
      "Bash(git add *)",
      "Bash(git commit *)",
      "Bash(git checkout *)",
      "Bash(git status)",
      "Bash(git diff *)",
      "Bash(git log *)",
      "Bash(git branch *)",
      "Bash(docker *)",
      "Bash(bash *)"
    ],
    "ask": [
      "Bash(git push *)",
      "Bash(git push)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(sudo *)"
    ]
  }
}
```

ポイントは3つある。

- **`Read` はパス制限なしで許可する。** 読み取りは副作用がないため安全である
- **`Edit` / `Write` はプロジェクトディレクトリに限定する。** システムファイルへの誤書き込みを防ぐ
- **`git push` は `ask` に入れる。** リモートへの変更反映は常に確認すべきである


## 注意すべき落とし穴

### Read/Edit の deny は Bash を止めない

`Read(//.env)` を deny にしても、`Bash(cat .env)` は防げない。\
機密ファイルの保護には、Bash パターンも合わせて deny に登録するか、OS レベルのサンドボックスで補完する必要がある。

### Bash パターンの構造的な限界

Bash パターンは引数の順序やシェル変数の展開を考慮しない。

```bash
# このパターンは...
"Bash(curl http://example.com/*)"

# 以下にはマッチしない
curl -X GET http://example.com/api    # オプションが先頭
curl $URL                             # 変数展開
```

ネットワークアクセスの制御は Bash パターンではなく、`Bash(curl *)` を deny にして `WebFetch(domain:...)` で管理するのが安全である。

### CLAUDE.md では権限を制御できない

CLAUDE.md はプロンプトへの指示であり、ツール実行の権限制御には影響しない。\
「このフォルダは編集してよい」と CLAUDE.md に書いても、確認ダイアログはスキップされない。\
権限の事前承認は必ず `settings.json` の `permissions.allow` で行う。


## 高度な制御 — PreToolUse フックとサンドボックス

![高度な制御の概念図](images/advanced-control.png)

`settings.json` のパターンマッチングでは対応しきれないケースには、2つの追加防御手段がある。


### PreToolUse フック

ツール実行前にシェルスクリプトで検査し、終了コード `0` で許可、`2` でブロックできる。

```bash
#!/bin/bash
# ~/.claude/hooks/pre-tool-use.sh
if echo "$INPUT" | grep -q "rm -rf /"; then
  echo "Blocked: dangerous rm command" >&2
  exit 2  # ブロック
fi
exit 0     # 許可
```

パターンマッチングの限界（変数展開やオプション順序の違い）を補完する手段として有効である。


### サンドボックスとの併用

権限システムとサンドボックスは異なる層で動作し、二重防御として機能する。

| 層 | 問い | 手段 |
| --- | --- | --- |
| 権限システム | このツールを実行すべきか？ | settings.json のルール |
| サンドボックス | 実行時に何にアクセスできるか？ | OS レベルの制限（macOS: Seatbelt、Linux: bubblewrap） |

言い換えると、権限システムは「門番」、サンドボックスは「壁」である。\
門番をすり抜けても壁がある、という構造が安全性を高める。


## エンタープライズ・チーム向けの管理

チーム全体で統一ポリシーを適用する場合は `managed-settings.json` を使う。\
個別ユーザーによる上書きができない強制設定である。

| OS | パス |
| --- | --- |
| Linux / WSL | `/etc/claude-code/managed-settings.json` |
| Windows | `C:\ProgramData\ClaudeCode\managed-settings.json` |

プロジェクト固有の設定は `.claude/settings.json`（コミット対象）で共有し、個人の差分は `.claude/settings.local.json`（`.gitignore` に追加）で管理する。


## まとめ

- **deny ルールは絶対。** allow より常に優先される。破壊的コマンドは必ず deny に登録する
- **`acceptEdits` + `settings.json` が実用的。** ファイル編集は自動承認し、Bash は頻出コマンドを allow に登録する
- **`Read` は制限なしで allow にしてよい。** 読み取りは副作用がない
- **`git push` は `ask` に入れる。** リモートへの反映は常に確認すべきである
- **CLAUDE.md では権限を制御できない。** 権限設定は必ず `settings.json` で行う
- **Bash パターンの限界を理解する。** ネットワーク制御は WebFetch のドメイン制限を使う


## 参考リンク

- [Configure permissions - Claude Code Docs](https://code.claude.com/docs/en/permissions)
- [Automate workflows with hooks](https://code.claude.com/docs/en/hooks-guide)
- [Sandboxing - Claude Code Docs](https://code.claude.com/docs/en/sandboxing)
- [Best Practices for Claude Code](https://code.claude.com/docs/en/best-practices)
- [Enterprise configuration](https://support.claude.com/en/articles/12622667-enterprise-configuration)
- [Claude Code settings.json: Complete config guide](https://www.eesel.ai/blog/settings-json-claude-code)
- [A complete guide to Claude Code permissions](https://www.eesel.ai/blog/claude-code-permissions)
- [YOLO Mode: Permission configuration](https://dev.to/rajeshroyal/yolo-mode-when-youre-tired-of-claude-asking-permission-for-everything-2daf)
- [Claude Code Security Best Practices](https://www.backslash.security/blog/claude-code-security-best-practices)
- [trailofbits/claude-code-config](https://github.com/trailofbits/claude-code-config)
