# Anytime Markdown 改善計画

更新日: 2026-03-20


## 仮定一覧

| # | 仮定 | 影響度 |
| --- | --- | --- |
| A1 | 主要ユーザーはデスクトップ（Web / VS Code）が中心であり、モバイルは補助的な利用 | 高 |
| A2 | ユーザーの技術レベルは中〜上級（Markdown に馴染みがある） | 中 |
| A3 | 色覚多様性への対応は AA 準拠を基準とし、AAA は将来課題とする | 中 |
| A4 | VS Code 拡張は Webview の CSP 制約によりコード分割が制限される | 低 |
| A5 | ExplorerPanel は VS Code 拡張専用であり、Web アプリでは使用されない | 低 |


## 現状サマリー

Anytime Markdown は Tiptap ベースのリッチ Markdown エディタで、Web / VS Code / Android の 3 プラットフォームに対応している。\
`editor-core` パッケージにエディタ機能を集約し、各プラットフォームが共有する設計である。

**【Designer】** UI は機能豊富だが、設定・メニューが分散しており情報設計に改善余地がある。\
モバイル対応はレスポンシブ breakpoint で基本的に処理されているが、タッチターゲットサイズや初回体験に課題がある。

**【A11y】** キーボード操作（ツールバーの roving tabindex）やスクリーンリーダー対応（`aria-label`、`aria-live`）は基盤が整っている。\
一方、色コントラスト（スクロールバー、インラインコード、見出し背景）に WCAG AA 未達の箇所がある。

**【Engineer】** TypeScript の型安全性、テスト基盤（Jest 56 スイート / 727 テスト）、CI/CD（日次ビルド + CodeQL + SonarCloud）は良好である。\
重い図表ライブラリの動的インポート未対応、`ExplorerPanel`（1466行）の巨大化、`localStorage` エラーハンドリングの不足が主な技術負債である。


## 課題一覧


### 高優先

| # | 課題 | カテゴリ | 担当 |
| --- | --- | --- | --- |
| H1 | スクロールバーの色コントラストが WCAG AA 未達 | A11y | A11y |
| H2 | インラインコードの色コントラストが WCAG AA 未達（ライトモード） | A11y | A11y |
| H3 | モバイルのタッチターゲットサイズが不足（32px、推奨 44px） | A11y / UX | Designer + A11y |
| H4 | 図表ライブラリ（Mermaid / KaTeX）が動的インポートされていない | 性能 | Engineer |
| H5 | `localStorage` のエラーハンドリングが不十分（quota 超過時にサイレント失敗） | 信頼性 | Engineer |


### 中優先

| # | 課題 | カテゴリ | 担当 |
| --- | --- | --- | --- |
| M1 | 初回体験が最小限（Welcome テンプレートが 4 行のみ） | UX | Designer |
| M2 | ツールバーが `flexWrap: wrap` でモバイル時に高さが不安定 | UX | Designer + Engineer |
| M3 | スラッシュコマンドメニューの検索結果数がスクリーンリーダーに通知されない | A11y | A11y |
| M4 | ダイアログのフォーカス管理が不統一 | A11y | A11y |
| M5 | `ExplorerPanel.tsx` が 1466 行で保守性が低い | 保守性 | Engineer |
| M6 | Web アプリのテストカバレッジが不足（テストファイル 4 件のみ） | 信頼性 | Engineer |
| M7 | 見出し背景のコントラストが境界線上 | A11y | A11y |
| M8 | 無効化されたボタンに理由が表示されない | UX | Designer |


### 低優先

| # | 課題 | カテゴリ | 担当 |
| --- | --- | --- | --- |
| L1 | ブロックラベルが CSS 疑似要素で実装されている（DOM 非存在） | A11y | A11y + Engineer |
| L2 | `as any` 型アサーションが 6 箇所 | 保守性 | Engineer |
| L3 | AWS SDK のバージョンが古い（3.1004.0） | セキュリティ | Engineer |
| L4 | `next-auth` v4 がメンテナンスモード | 保守性 | Engineer |
| L5 | CI にバンドルサイズ分析がない | 性能 | Engineer |
| L6 | アウトラインパネル幅がモバイルで固定 220px | UX | Designer |
| L7 | Jest の `maxWorkers` が editor-core と web-app で不統一 | 保守性 | Engineer |


## 改善案


### H1: スクロールバーの色コントラスト修正

**担当:** A11y\
**工数:** S\
**期待効果:** WCAG 1.4.3 Contrast (AA) 準拠

**現状:** ライトモード `rgba(0,0,0,0.15)`（約 1:3.6）、ダークモード `rgba(255,255,255,0.2)`（約 1:3.5）で AA 基準 3:1 未達。

**改善案:**

- ライトモード: `rgba(0,0,0,0.4)` 以上に変更
- ダークモード: `rgba(255,255,255,0.5)` 以上に変更
- hover 時もコントラスト比を維持

**対象ファイル:** `editorStyles.ts`

> **【Designer レビュー】推奨:** 視覚的な主張が強くなりすぎないよう、0.4〜0.45 の範囲で調整する。\
> **【Engineer レビュー】質問なし。** XS の変更で実装可能。


### H2: インラインコード色コントラスト修正

**担当:** A11y\
**工数:** S\
**期待効果:** WCAG 1.4.3 準拠

**現状:** ライトモードのインラインコードが `theme.palette.error.main`（`#d32f2f`）を背景 `#F1F5F9` 上で使用。\
コントラスト比は約 3.8:1 で AA 基準 4.5:1 未達。

**改善案:** `#b71c1c` または `#c62828` に変更し、4.5:1 以上を確保する。

**対象ファイル:** `codeStyles.ts`


### H3: モバイルのタッチターゲットサイズ拡大

**担当:** Designer + A11y\
**工数:** M\
**期待効果:** WCAG 2.5.8 Target Size 準拠、モバイル操作性向上

**現状:** `SIDE_TOOLBAR_ICON_SIZE = 32px`、`IconButton size="small"` = 36px。

**改善案:**

- `SIDE_TOOLBAR_ICON_SIZE` をモバイル時 44px に変更
- ToggleButton にモバイル時のパディング増加: `sx={{ minHeight: { xs: 44, md: 30 } }}`
- ツールバーボタン全体のタッチ領域を 44px 以上に確保

**対象ファイル:** `dimensions.ts`, `EditorToolbar.tsx`, `EditorSideToolbar.tsx`


### H4: 図表ライブラリの動的インポート

**担当:** Engineer\
**工数:** M\
**期待効果:** 初期バンドルサイズ削減（推定 150KB+）

**現状:** `mermaid`（11.12.3）と `katex`（0.16.38）が静的インポートされ、初期バンドルに含まれる。

**改善案:**

- `DiagramBlock` / `MathBlock` を `React.lazy()` でラップ
- Suspense fallback にローディングスピナーを表示
- VS Code 拡張は CSP 制約により単一バンドル維持（影響なし）

**対象ファイル:** `CodeBlockNodeView.tsx`, 図表レンダラーコンポーネント群

> **【A11y レビュー】推奨:** ローディング中の Suspense fallback に `aria-busy="true"` と `aria-label` を付与する。


### H5: localStorage エラーハンドリング強化

**担当:** Engineer\
**工数:** S\
**期待効果:** データ喪失リスクの軽減、ユーザーへの適切な通知

**現状:** 8 箇所の `localStorage` 呼び出しで `catch { /* */ }` によるサイレント失敗。

**改善案:**

- `QuotaExceededError` 検出時にトースト通知を表示
- 保存失敗時にリトライまたは古いデータの自動クリーンアップ
- localStorage ラッパー関数を作成し、一元管理

**対象ファイル:** `useEditorSettings.ts`, `useMarkdownEditor.ts`, `useSourceMode.ts`


### M1: 初回体験の改善

**担当:** Designer\
**工数:** S\
**期待効果:** 新規ユーザーの機能発見率向上

**現状:** Welcome テンプレートは 4 行で、スラッシュコマンドの 1 例のみ。

**改善案:**

- Welcome テンプレートにモード切替、パネル操作の簡潔な説明を追加（10 行以内）
- スラッシュコマンドの入力例を 2〜3 種類に増やす（`/table`, `/mermaid` など）

**対象ファイル:** `welcome.md`, `welcome-en.md`

> **【A11y レビュー】推奨:** 追加テキストのコントラストと構造（見出しレベル）を適切に設定する。


### M2: モバイルツールバーの高さ安定化

**担当:** Designer + Engineer\
**工数:** S\
**期待効果:** モバイル表示の安定化

**改善案:** `flexWrap: "wrap"` を `flexWrap: "nowrap"` + `overflowX: "auto"` に変更し、横スクロールで対応する。

**対象ファイル:** `EditorToolbar.tsx`


### M3: スラッシュコマンドメニューのライブリージョン追加

**担当:** A11y\
**工数:** S\
**期待効果:** WCAG 4.1.3 Status Messages 準拠

**改善案:** フィルタ結果の件数をライブリージョンで通知する。

```tsx
<Typography role="status" aria-live="polite" aria-atomic="true">
  {filteredItems.length > 0
    ? `${filteredItems.length} 件`
    : t("slashCommandNoResults")}
</Typography>
```

**対象ファイル:** `SlashCommandMenu.tsx`


### M4: ダイアログフォーカス管理の統一

**担当:** A11y\
**工数:** S\
**期待効果:** WCAG 2.4.3 Focus Order 準拠

**改善案:** MUI Dialog の `autoFocus` と `disableRestoreFocus` を統一的に設定する。\
`ConfirmDialog` の二重 `autoFocus` を修正する。

**対象ファイル:** `ConfirmDialog.tsx`, `EditDialogWrapper.tsx`


### M5: ExplorerPanel の分割

**担当:** Engineer\
**工数:** L\
**期待効果:** 保守性向上、テスタビリティ改善

**改善案:**

- TreeView、FileOperations、GitHistory を個別コンポーネントに抽出
- ロジックをカスタムフックに分離
- 各コンポーネントにユニットテストを追加

**対象ファイル:** `ExplorerPanel.tsx` → 3〜4 ファイルに分割


### M6: Web アプリのテストカバレッジ向上

**担当:** Engineer\
**工数:** XL\
**期待効果:** 回帰バグの早期検出、リファクタリング安全性向上

**改善案:**

- CMS コンポーネント（`EditBody`, `CardAreaPanel`）のテスト追加
- ファイル操作（`WebFileSystemProvider`）のテスト追加
- カバレッジ目標を 60% に設定

**対象ファイル:** `packages/web-app/src/__tests__/` 配下に新規作成


### M8: 無効ボタンの理由表示

**担当:** Designer\
**工数:** S\
**期待効果:** ユーザーの混乱を軽減

**改善案:** ソースモード時や比較モード時に無効化されるボタンのツールチップを「ソースモードでは利用できません」等に変更する。

**対象ファイル:** `EditorToolbar.tsx`

> **【A11y レビュー】必須:** `disabled` 状態のボタンでもツールチップが表示されるよう、`span` ラッパーで Tooltip を有効化する。


## アクセシビリティ監査結果


### WCAG 2.2 AA 準拠状況

| 基準 | 状態 | 備考 |
| --- | --- | --- |
| 1.1.1 非テキストコンテンツ | 準拠 | 画像に `alt` 属性、アイコンに `aria-label` あり |
| 1.3.1 情報と関係性 | 一部未達 | ブロックラベルが CSS 疑似要素（L1） |
| 1.4.3 コントラスト | 未達 | スクロールバー（H1）、インラインコード（H2）、見出し背景（M7） |
| 2.1.1 キーボード | 準拠 | Roving tabindex パターン実装済み |
| 2.4.3 フォーカス順序 | 一部未達 | ダイアログのフォーカス管理が不統一（M4） |
| 2.4.7 フォーカスの可視化 | 準拠 | MUI デフォルトのフォーカスリング使用 |
| 2.5.8 ターゲットサイズ | 未達 | モバイルで 32px（H3） |
| 3.3.1 エラーの特定 | 一部未達 | `aria-describedby` の不統一 |
| 4.1.3 ステータスメッセージ | 一部未達 | スラッシュメニューの結果件数未通知（M3） |


### 良好な実装

- ツールバーの WAI-ARIA Toolbar パターン（roving tabindex）
- `StatusBar` の `aria-live="polite"` によるステータス通知
- スライダーの `aria-label` + `aria-valuetext`
- `prefers-reduced-motion` へのアニメーション対応（`OutlinePanel`, `CodeBlockFrame`）
- `SlashCommandMenu` の `role="menu"` / `role="menuitem"` パターン


## 改善ロードマップ


### Quick Win（1〜2 日）

| # | 改善 | 工数 |
| --- | --- | --- |
| H1 | スクロールバー色コントラスト修正 | S |
| H2 | インラインコード色コントラスト修正 | S |
| M3 | スラッシュメニューのライブリージョン追加 | S |
| M4 | ダイアログフォーカス管理統一 | S |
| L2 | `as any` 型アサーションを型スタブに置換 | XS |
| L7 | Jest `maxWorkers` 統一 | XS |


### 短期（1〜2 週間）

| # | 改善 | 工数 |
| --- | --- | --- |
| H3 | モバイルタッチターゲットサイズ拡大 | M |
| H4 | 図表ライブラリの動的インポート | M |
| H5 | localStorage エラーハンドリング強化 | S |
| M1 | Welcome テンプレート改善 | S |
| M2 | モバイルツールバー安定化 | S |
| M8 | 無効ボタンの理由表示 | S |
| L5 | CI にバンドルサイズ分析追加 | S |


### 中期（1〜2 ヶ月）

| # | 改善 | 工数 |
| --- | --- | --- |
| M5 | ExplorerPanel 分割リファクタリング | L |
| M6 | Web アプリのテストカバレッジ向上 | XL |
| L1 | ブロックラベルの DOM 実装化 | M |
| L3 | AWS SDK バージョン更新 | S |
| L4 | next-auth v5 移行検討 | M |
| L6 | アウトラインパネルのモバイル対応 | S |


## リスクと未解決課題

| リスク | 影響度 | 対策 |
| --- | --- | --- |
| H4 の動的インポートで VS Code 拡張に影響 | 中 | Webpack の `LimitChunkCountPlugin` により拡張側は影響なし。\Web アプリのみ適用。 |
| M5 の ExplorerPanel 分割で既存機能に回帰バグ | 高 | 分割前にスナップショットテストを追加し、分割後に E2E テストで検証。 |
| L4 の next-auth v5 移行で認証フローが変更 | 中 | v5 の breaking changes を事前調査し、移行計画を別途作成。 |
| モバイルタッチターゲット拡大でデスクトップの密度が低下 | 低 | レスポンシブ breakpoint でモバイル/デスクトップを分離。 |

> **【Designer + A11y 間の意見対立】**\
> ブロックラベル（H1, P, Quote 等）の表示方法について、Designer は「常時表示は視覚的ノイズになる」、A11y は「hover のみでは keyboard/screen reader ユーザーがアクセスできない」と対立した。\
> **推奨案:** デフォルトは現状維持（hover 表示）とし、設定パネルに「ブロックラベルを常時表示」トグルを追加する。\
> これにより、キーボードユーザーや支援技術ユーザーは常時表示を有効にでき、一般ユーザーはクリーンな表示を維持できる。


## 使用スキル

| エージェント | 使用スキル |
| --- | --- |
| Designer | `superpowers:brainstorming` |
| A11y | `code-review-checklist` |
| Engineer | `superpowers:requesting-code-review` |
