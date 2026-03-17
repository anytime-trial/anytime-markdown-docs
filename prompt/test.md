Phase 0: 環境整備（一度だけ、以後メンテナンス）
  ├── CLAUDE.md を50行以下に圧縮
  ├── .claude/rules/ に関心事ごとのルールを配置
  ├── .claude/skills/ にワークフローSkillを配置
  └── .claude/settings.json にdenyルールを設定

Phase 1: 調査（PLAN MODE）
  └── コードベースを深く調査 → research.md

Phase 2: 計画（PLAN MODE）
  └── plan.md 作成 → アノテーションサイクル × 1-6回 → Todoリスト化

Phase 3: 実装（--dangerously-skip-permissions）
  └── git checkpoint → 計画に基づいて自律実行

Phase 4: 検証
  └── diff確認 → テスト → 必要なら修正
  

# プロジェクト概要

RustによるgRPC APIサーバー。sqlxでDB接続。

# 優先順位

正しさ > 保守性 > パフォーマンス > 簡潔さ

# ビルド・テスト

cargo fmt --check && cargo clippy -- -D warnings && cargo test --workspace

# 禁止事項

- unwrap() / expect() は本番コードで使用禁止
- unsafe ブロック追加禁止
- 既存のpublic APIシグネチャ変更禁止

# 参照

See @README.md for architecture overview