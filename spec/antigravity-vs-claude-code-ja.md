# Google Antigravity vs Claude Code 比較調査

更新日: 2026-03-21


## 概要

Google Antigravity（2025年11月発表）は、Gemini 3 をベースとしたエージェント型 AI 開発プラットフォーム。\
VS Code のフォークとして構築された IDE であり、Claude Code のターミナルベースのアプローチとは設計思想が異なる。

| 項目 | Google Antigravity | Claude Code |
| --- | --- | --- |
| 発表時期 | 2025年11月 | 2025年初頭 |
| アーキテクチャ | エージェントファースト IDE | ターミナルファースト CLI |
| ベース | VS Code フォーク | 独自 CLI |
| 主要モデル | Gemini 3.1 Pro / Flash | Claude Opus 4.6 / Sonnet 4.6 |
| 開発元 | Google | Anthropic |


## 設計思想の違い

| 観点 | Antigravity | Claude Code |
| --- | --- | --- |
| エージェントの自律性 | 高い（最小限のチェックポイントで自律判断） | ヒューマンインザループ（変更前に承認） |
| ワークフロー | IDE 内で完結（エディタ + ターミナル + ブラウザ） | 既存のワークフローに統合 |
| 操作対象 | エディタ・ターミナル・ブラウザを横断 | ファイルシステム・シェル・Git |
| 複数エージェント | Mission Control で並列エージェント管理 | サブエージェント・Agent SDK |
| 安全性 | 自律的で破壊的コマンドのリスク報告あり | 権限システム + サンドボックス |


## 機能比較

| 機能 | Antigravity | Claude Code |
| --- | --- | --- |
| コード生成 | o | o |
| ファイル管理 | o | o |
| テスト実行 | o | o |
| Git 操作 | o | o |
| ブラウザ統合 | o（組み込みブラウザ） | o（MCP / Playwright） |
| MCP サーバー | o（2026年初頭に追加） | o |
| タブ補完 | o（無制限） | x |
| マルチモデル対応 | o（Gemini + Claude + GPT-OSS） | x（Claude モデルのみ） |
| IDE 統合 | o（独自 IDE） | o（VS Code 拡張等） |
| カスタムスキル | 不明 | o |
| 権限システム | 限定的 | o（allow / ask / deny） |
| エンタープライズ | 未提供（2026年3月時点） | o（SSO / SCIM / 監査ログ / HIPAA） |


## ベンチマーク比較

| ベンチマーク | Antigravity | Claude Code |
| --- | --- | --- |
| SWE-bench Verified | 76.2% | 76.8%〜80.9%（Claude Opus 4.5/4.6） |
| WebDev Arena | 1487 Elo | 非公開 |
| Terminal-Bench 2.0 | 54.2% | 非公開 |

> SWE-bench は実際の GitHub Issue を解決する能力を測定するベンチマーク。\
> Claude Opus が最高スコア（76.8%〜80.9%）を保持している。


## 料金比較

### Antigravity

| プラン | 月額 | 内容 |
| --- | --- | --- |
| Free | $0 | 全モデルアクセス（週単位のレート制限あり） |
| Pro | $20 | 5時間ごとにクォータ更新、優先アクセス |
| Ultra | $249.99 | 最大クォータ |
| 追加クレジット | $25 / 2,500クレジット | 不足時に購入 |

> Free プランでも Claude Opus 4.6、Gemini 3.1 Pro、GPT-OSS が利用可能。\
> ただし、集中的なコーディングでは2〜3時間で制限に達するとの報告がある。

### Claude Code

| プラン | 月額 | 内容 |
| --- | --- | --- |
| Pro | $20 | Sonnet 4.5 + Opus（制限あり） |
| Max 5x | $100 | Pro の5倍の使用量 |
| Max 20x | $200 | Pro の20倍の使用量 |
| API | 従量課金 | $3〜$25 / 100万トークン |

> Claude Code は Claude モデルのみ。\
> Antigravity は無料で複数モデルを使える点がコスト面での優位性。


## 2026年3月時点の課題

### Antigravity

- **安全性の懸念:** 複数の開発者がエージェントの自律的な破壊的コマンド（ドライブ消去、プロジェクトファイル破損）を報告
- **クォータ変更:** 3月11日にクレジット制に移行し、Pro ユーザーに遅延が発生
- **エンタープライズ未対応:** SSO・監査ログ・HIPAA 等のセキュリティ認証が未整備

### Claude Code

- **使用量制限:** Max プランでもワークフロー途中で制限に達するケースの報告あり
- **単一モデル:** Claude モデルのみで、モデル切替の柔軟性がない
- **コスト:** 大量使用時は Max（$100〜$200/月）が必要


## 選定ガイド

| 重視する点 | 推奨 |
| --- | --- |
| 無料で試したい | Antigravity（Free プランで全モデル利用可能） |
| 安全性・権限管理 | Claude Code（権限システム + サンドボックス） |
| エンタープライズ利用 | Claude Code（SSO / SCIM / 監査ログ） |
| 複数モデルの比較 | Antigravity（Gemini + Claude + GPT-OSS） |
| ターミナル中心のワークフロー | Claude Code |
| ビジュアル IDE が好み | Antigravity |
| 既存の VS Code 設定を活かしたい | Claude Code（VS Code 拡張として統合） |
| 自律的なエージェントに任せたい | Antigravity（ただしリスクあり） |
| ヒューマンインザループで進めたい | Claude Code |


## 参考資料

- [Google Antigravity 公式](https://antigravity.google/)
- [Build with Google Antigravity - Google Developers Blog](https://developers.googleblog.com/build-with-google-antigravity-our-new-agentic-development-platform/)
- [Google Antigravity Review 2026](https://vibecoding.app/blog/google-antigravity-review)
- [Claude Code vs Antigravity: Which is Better? - DataCamp](https://www.datacamp.com/blog/claude-code-vs-antigravity)
- [Google Antigravity vs Claude Code - Augment Code](https://www.augmentcode.com/tools/google-antigravity-vs-claude-code)
- [AI dev tool power rankings - LogRocket](https://blog.logrocket.com/ai-dev-tool-power-rankings/)
- [Google Antigravity Pricing](https://antigravity.google/pricing)
- [Claude Code Pricing Guide](https://www.ksred.com/claude-code-pricing-guide-which-plan-actually-saves-you-money/)
