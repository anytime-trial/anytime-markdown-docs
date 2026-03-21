# リファクタリング提案書

更新日: 2026-03-21


## 概要

プロジェクト全体を調査し、リファクタリング候補を19件特定した。\
動作リスク・即効性・保守性の観点で高・中・低の3段階に分類する。


## 高優先度（動作リスク・即効性あり）


### 1. StatusBar の DOM 探索を ref 経由に変更

**対象:** `editor-core/src/components/StatusBar.tsx`

**問題:**\
`document.querySelector("textarea[aria-label]")` でソースモードの `<textarea>` を探索している。\
`aria-label` の変更や `<textarea>` が複数存在するケースで壊れる。

**対応方針:**\
`SourceModeEditor` から ref を上位に公開し、`StatusBar` は ref 経由でアクセスする。


### 2. localStorage.setItem モンキーパッチの除去

**対象:** `web-app/src/app/markdown/page.tsx`

**問題:**\
`localStorage.setItem` をモンキーパッチして dirty 判定に使用している。\
副作用がグローバルに波及するアンチパターン。

```typescript
const origSetItem = localStorage.setItem.bind(localStorage);
localStorage.setItem = (key: string, value: string) => { ... };
```

**対応方針:**\
エディタの `onUpdate` コールバックで dirty フラグを管理する方式に変更する。


### 3. GitHub API 関数・型定義の重複統合

**対象:**

- `web-app/src/lib/githubApi.ts`
- `web-app/src/components/explorer/helpers.ts`
- `web-app/src/components/GitHubRepoBrowser.tsx`

**問題:**\
`fetchDirEntries`、`fetchCommits`、`fetchBranches` 等の fetch 関数が `githubApi.ts` と `explorer/helpers.ts` の2箇所に定義されている。\
`TreeEntry` 型も3箇所で定義され、`type` フィールドの値が `"file"|"dir"` と `"blob"|"tree"` で不一致。\
`GitHubRepoBrowser.tsx` はどこからもインポートされておらず、デッドコードの可能性がある。

**対応方針:**

- `lib/githubApi.ts` に関数・型を統一
- `explorer/helpers.ts` のフィルタリング・ソートロジックを `githubApi.ts` に移行
- `GitHubRepoBrowser.tsx` の使用状況を確認し、未使用なら削除


### 4. showError() の未使用箇所を統一

**対象:**

- `vscode-extension/src/utils/errorHelpers.ts`（`showError()` 定義済み）
- `vscode-extension/src/providers/MarkdownEditorProvider.ts`（5箇所）
- `vscode-extension/src/providers/ChangesProvider.ts`（10箇所）

**問題:**\
`errorHelpers.ts` に `showError()` ヘルパーが存在するにもかかわらず、`MarkdownEditorProvider.ts` と `ChangesProvider.ts` では `err instanceof Error ? err.message : String(err)` を直書きしている。\
15箇所の重複。

**対応方針:**\
全箇所を `showError()` 呼び出しに統一する。


### 5. tiptap v3 TODO パッチの検証

**対象:** `editor-core/src/MarkdownEditorPage.tsx:5`

**問題:**\
`console.error` パッチに `// TODO: tiptap v3 で修正される見込み` というコメントがある。\
現在の tiptap は `3.20.0` で、既に v3 に移行済み。\
パッチが不要になっている可能性がある。

**対応方針:**\
パッチを外してテスト実行し、エラーが再発しなければ削除する。


## 中優先度（保守性改善）


### 6. PlantUML @start 判定ロジックの集約

**対象:**

- `editor-core/src/hooks/usePlantUmlRender.ts`
- `editor-core/src/hooks/useDiagramCapture.ts`
- `editor-core/src/hooks/useEditorFileOps.ts`

**問題:**\
3箇所で `code.match(/@start(uml|mindmap|wbs|json|yaml)/)` による同一判定と `@startuml\n...\n@enduml` でのラップ処理が重複。

**対応方針:**\
`utils/plantumlHelpers.ts` に `normalizePlantUmlCode(code: string): string` を追加し、3箇所から呼び出す。


### 7. Mermaid オフスクリーンコンテナ生成の集約

**対象:**

- `editor-core/src/hooks/useMermaidRender.ts`
- `editor-core/src/hooks/useDiagramCapture.ts`
- `editor-core/src/hooks/useEditorFileOps.ts`

**問題:**\
3箇所で以下の同一パターンが繰り返されている。

```typescript
container.style.position = "absolute";
container.style.left = "-9999px";
container.style.top = "-9999px";
document.body.appendChild(container);
```

**対応方針:**\
`createOffscreenContainer(id: string): HTMLDivElement` ユーティリティ関数に集約する。


### 8. EditorSettings 型の逆転依存を解消

**対象:**

- `editor-core/src/constants/colors.ts`
- `editor-core/src/useEditorSettings.ts`

**問題:**\
定数ファイル `colors.ts` がアプリ層の `useEditorSettings.ts` から `EditorSettings` 型をインポートしている。\
定数が実装に依存する逆転した構造。

**対応方針:**\
`EditorSettings` 型定義を `types/` ディレクトリに移動し、依存方向を正す。


### 9. index.ts の過剰エクスポート整理

**対象:** `editor-core/src/index.ts` および `editor-core/src/exports/`

**問題:**

- `MarkdownIcon` — 完全未使用のデッドコード
- `sanitizeMarkdown`、`preserveBlankLines`、diff 系関数 — 外部利用者なし
- `BUILTIN_TEMPLATES` — `getBuiltinTemplates(locale)` が正しい API で、固定 JA 版の定数は不要
- `exports/constants.ts`、`exports/extensions.ts`、`exports/types.ts` と `index.ts` で二重エクスポート

**対応方針:**

- 外部利用のないシンボルを `index.ts` から削除
- `MarkdownIcon` を完全削除
- サブパス exports との重複を解消


### 10. MermaidNodeView.tsx のファイル名とエクスポート名の不一致

**対象:** `editor-core/src/MermaidNodeView.tsx`

**問題:**\
ファイル名は `MermaidNodeView` だが、主要エクスポートは `CodeBlockNodeView`。\
`index.ts` でも `export { CodeBlockNodeView } from './MermaidNodeView'` となっており、ファイル検索時に混乱を招く。

**対応方針:**\
ファイルを `CodeBlockNodeView.tsx` にリネームする。


### 11. useConfirm インポートパスの混在

**対象:** editor-core 全体

**問題:**\
`@/hooks/useConfirm` と `../hooks/useConfirm` が混在している。

**対応方針:**\
プロジェクト全体でどちらかに統一する。


## 低優先度（整理・改善）


### 12. MarkdownEditorPage の props 整理

**対象:** `editor-core/src/MarkdownEditorPage.tsx`（537行、37 props）

**問題:**\
`hide*` フラグ9個を含む37個の props を持つゴッドコンポーネント。

**対応方針:**\
props を機能グループ（visibility / fileOps / externalContent / editorBehavior）のオブジェクト型に整理する。\
残存する `useEffect` を専用フックに切り出す。


### 13. diffHighlight.ts のアルゴリズム分離

**対象:** `editor-core/src/extensions/diffHighlight.ts`（580行）

**問題:**\
ProseMirror プラグイン定義とブロック差分計算アルゴリズム（LCS、セクション分割、デコレーション構築）が同居。

**対応方針:**\
`computeSemanticBlockDiff` 等のアルゴリズム部分を `diffHighlightAlgorithms.ts` に分離する。


### 14. gifEncoder.ts のモジュール分離

**対象:** `editor-core/src/utils/gifEncoder.ts`（575行）

**問題:**\
NeuQuant ニューラルネット色量子化（190行）、LZW 圧縮（100行）、GIF フレーム書き出しの3つの異なる技術が同居。

**対応方針:**\
`NeuQuant` クラスを別ファイルに分離する。


### 15. useEditorConfig.ts のファイルドロップ処理切り出し

**対象:** `editor-core/src/hooks/useEditorConfig.ts`（416行）

**問題:**\
ドロップ / ペースト処理で `if → forEach → FileReader.onload → if` の4段ネストがある。\
コンテキストメニュー、キーボード設定も同居。

**対応方針:**\
ファイルドロップ処理を `useEditorFileDrop.ts` に切り出す。


### 16. NodeView / Hook ファイルのディレクトリ整理

**対象:**

- `editor-core/src/ImageNodeView.tsx`、`MermaidNodeView.tsx`、`TableNodeView.tsx` — `src/` 直下
- `editor-core/src/useEditorSettings.ts`、`useMarkdownEditor.ts` — `hooks/` の外

**問題:**\
`components/` や `hooks/` ディレクトリが存在するにもかかわらず、一部のファイルが `src/` 直下に混在。

**対応方針:**\
NodeView を `components/` 配下、フックを `hooks/` 配下に移動する。\
インポートパスの一括更新が必要。


### 17. API routes の console.warn を適切なレベルに変更

**対象:**

- `web-app/src/app/api/github/content/route.ts`
- `web-app/src/app/api/github/commits/route.ts`

**問題:**\
正常系のログに `console.warn` を使用している。

**対応方針:**\
`console.info` に変更、または削除する。


### 18. PANEL_HEADER_MIN_HEIGHT 重複の解消

**対象:**

- `editor-core/src/constants/dimensions.ts`
- `web-app/src/components/explorer/types.ts`

**問題:**\
同一の定数 `PANEL_HEADER_MIN_HEIGHT = 40` が別パッケージで独立定義されている。

**対応方針:**\
`web-app` 側を `@anytime-markdown/editor-core` からのインポートに変更する。


### 19. エラー処理パターンの統一

**対象:** editor-core 全体

**問題:**\
以下の4パターンが混在している。

- `console.error` でログのみ（UI に伝達しない）
- `console.warn` でログのみ
- `.catch(() => { /* comment */ })` で完全無視
- `.catch(() => { フォールバック処理 })` で代替処理

**対応方針:**\
エラー処理の方針を定め、プロジェクト全体で統一する。


## 推奨実施順序

1. 高優先度 #1〜#5（動作リスクの解消、即効性）
2. 中優先度 #6〜#7（重複コード集約、TDD で安全に実施可能）
3. 中優先度 #8〜#11（依存関係・命名の整理）
4. 低優先度 #12〜#19（構造改善、段階的に実施）
