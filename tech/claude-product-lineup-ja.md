# Claude 商品群の仕様比較

更新日: 2026-03-21


## 概要

Anthropic が提供する Claude 関連製品は、大きく3つのカテゴリに分かれる。

| カテゴリ | 製品 | 対象 |
| --- | --- | --- |
| コンシューマー | Claude（Web / Desktop / Mobile） | 一般ユーザー |
| 開発者向けツール | Claude Code（CLI / Desktop） | ソフトウェア開発者 |
| プラットフォーム | Claude API | アプリケーション開発者 |


## 製品別の機能比較

| 機能 | Claude Web/Desktop | Claude Code CLI | Claude Code Desktop | Claude API |
| --- | --- | --- | --- | --- |
| インターフェース | ブラウザ / アプリ | ターミナル | デスクトップアプリ | HTTP API |
| 会話 / チャット | o | o | o | o |
| ファイル読み書き | x | o | o | o（アプリ実装次第） |
| コマンド実行 | x | o | o | x |
| Web 検索 | o（Pro 以上） | o | o | x |
| MCP サーバー | o | o | o | x |
| カスタムスキル | x | o | o | x |
| バックグラウンドタスク | o（Cowork） | o（サブエージェント） | o | x |
| スケジュール実行 | o（Cowork `/schedule`） | o（`/loop`） | o | x |
| Git 操作 | x | o | o | x |
| IDE 統合 | x | o（VS Code 等） | x | x |
| マルチエージェント | x | o（Agent SDK） | x | o（Agent SDK） |


## Cowork（Claude Desktop のエージェント機能）

Claude Desktop に搭載されたバックグラウンドエージェント機能。\
Claude Code と同じエージェントアーキテクチャを、ターミナル不要で利用できる。

| 項目 | 内容 |
| --- | --- |
| 対象 | Pro 以上の有料プラン |
| プラットフォーム | Windows / macOS |
| ファイルアクセス | 明示的に許可したフォルダのみ |
| スケジュール | `/schedule` で定期実行可能 |
| サブエージェント | 複雑なタスクを分割し並列実行 |
| 指示の永続化 | グローバル指示 / フォルダ別指示 |

> Claude Code CLI との違い: Cowork はセッションが短く、ガードレールが多い。\
> マルチエージェントのオーケストレーションは CLI と Agent SDK でのみ利用可能。


## サブスクリプションプラン

| プラン | 月額 | Claude Code | Cowork | モデル | 使用量 |
| --- | --- | --- | --- | --- | --- |
| Free | $0 | x | x | Sonnet 4.5 | 制限あり |
| Pro | $20 | o | o | Sonnet 4.5 + Opus（制限あり） | 標準 |
| Max 5x | $100 | o | o | 全モデル | Pro の5倍 |
| Max 20x | $200 | o | o | 全モデル | Pro の20倍 |
| Team | $25〜$150/席 | o（Premium 席） | o | 全モデル | 席種による |
| Enterprise | カスタム | o | o | 全モデル | カスタム |

> Team プランの Standard 席（$25〜$30/月）には Claude Code は含まれない。\
> Premium 席（$150/月）で Claude Code が利用可能。


## API 料金（従量課金）

サブスクリプションとは **完全に別の課金体系**。\
API キーを取得し、使用量に応じて課金される。

| モデル | 入力（100万トークン） | 出力（100万トークン） |
| --- | --- | --- |
| Opus 4.5 | $5 | $25 |
| Sonnet 4.5 | $3 | $15 |
| Haiku 4.5 | $1 | $5 |

### コスト削減オプション

| オプション | 割引率 | 条件 |
| --- | --- | --- |
| Batch API | 50%割引 | 非同期処理（即時応答不要） |
| Prompt Caching（5分） | 入力の90%割引 | 書き込み 1.25倍、1回の再利用で元が取れる |
| Prompt Caching（1時間） | 入力の90%割引 | 書き込み 2倍、2回の再利用で元が取れる |


## サブスクリプション vs API の使い分け

| 用途 | 推奨 | 理由 |
| --- | --- | --- |
| 日常の開発作業 | Pro / Max サブスクリプション | 定額で Claude Code が使える |
| CI/CD での自動実行 | API | GitHub Actions 等から呼び出す場合は API キーが必要 |
| 自社アプリへの組み込み | API | アプリケーションから Claude を呼び出すには API が必須 |
| 大量のバッチ処理 | API（Batch） | 50%割引で非同期処理 |
| チームでの共同利用 | Team / Enterprise | 管理機能・SSO・監査ログ |


## 製品間の関係

```
Claude (Web/Desktop/Mobile)
├── チャット・ファイルアップロード・Web検索
├── Cowork（バックグラウンドエージェント）
└── Projects（プロンプト管理）

Claude Code
├── CLI（ターミナルベース）
│   ├── スキル・カスタムコマンド
│   ├── MCP サーバー
│   ├── サブエージェント・マルチエージェント
│   └── IDE 統合（VS Code 等）
└── Desktop（GUI ベース）
    ├── 組み込みブラウザ
    └── ビジュアルフィードバック

Claude API
├── Messages API（チャット）
├── Batch API（非同期バッチ）
├── Agent SDK（エージェント構築）
└── Tool Use（関数呼び出し）
```

> すべての製品は同じ Claude モデル（Opus / Sonnet / Haiku）を使用する。\
> 知能や回答品質に差はなく、違いは「ツール・機能・インターフェース」のみ。


## 参考資料

- [Plans & Pricing | Claude by Anthropic](https://claude.com/pricing)
- [Pricing - Claude API Docs](https://platform.claude.com/docs/en/about-claude/pricing)
- [Using Claude Code with your Pro or Max plan](https://support.claude.com/en/articles/11145838-using-claude-code-with-your-pro-or-max-plan)
- [Claude Code Desktop](https://code.claude.com/docs/en/desktop)
- [Cowork Product Page](https://claude.com/product/cowork)
- [Claude, Claude API, and Claude Code: What's the Difference?](https://eval.16x.engineer/blog/claude-vs-claude-api-vs-claude-code)
- [Claude Code Pricing Guide](https://www.ksred.com/claude-code-pricing-guide-which-plan-actually-saves-you-money/)
