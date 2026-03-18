# 全コード リファクタリング調査レポート

更新日: 2026-03-18


## 調査対象

- `packages/editor-core/src/` — エディタコアライブラリ（155ファイル、約20,000 LOC）
- `packages/vscode-extension/src/` — VS Code 拡張（10ファイル、約2,500 LOC）
- `packages/web-app/src/` — Web アプリ（30ファイル、約4,000 LOC）


## サマリー

| カテゴリ | editor-core | vscode-extension | web-app |
| --- | --- | --- | --- |
| ゴッドファイル（400行超） | 16件 | 3件 | 2件 |
| 重複ロジック | 高（比較モード条件） | 高（エラー処理14箇所） | 中（fetch パターン） |
| Props 肥大化 | 3件（22/29/13 props） | - | 1件（26 useState） |
| 型安全性 | 良好 | `as unknown` 5箇所 | 良好 |
| テストカバレッジ | 52スイート/695テスト | なし | なし |


## editor-core


### ゴッドファイル

| ファイル | 行数 | 優先度 |
| --- | --- | --- |
| `extensions/diffHighlight.ts` | 517 | 高 |
| `components/InlineMergeView.tsx` | 496 | 高 |
| `components/FullscreenDiffView.tsx` | 489 | 高 |
| `utils/diffEngine.ts` | 451 | 高 |
| `MarkdownEditorPage.tsx` | 449 | 中 |
| `utils/sanitizeMarkdown.ts` | 442 | 中 |
| `components/MergeEditorPanel.tsx` | 439 | 高 |
| `components/EditorToolbar.tsx` | 435 | 高 |
| `components/SearchReplaceBar.tsx` | 417 | 中 |
| `components/OutlinePanel.tsx` | 388 | 中 |
| `hooks/useEditorFileOps.ts` | 372 | 中 |
| `hooks/useEditorConfig.ts` | 368 | 中 |
| `extensions/commentExtension.ts` | 362 | 中 |
| `components/EditorMenuPopovers.tsx` | 362 | 中 |


### 重複ロジック

**比較モード条件の重複（4ファイル）**

`DiagramBlock`, `MathBlock`, `RegularCodeBlock`, `HtmlPreviewBlock` で同じ条件パターンが繰り返されている。

```typescript
// 4ファイルで繰り返される条件
(props.isCompareLeft && !props.isCompareLeftEditable) ? false : ...
(props.isCompareLeft && !props.isCompareLeftEditable) ? null : ...
```

> 抽出候補: `isReadOnlyCompareLeft(isCompareLeft, isCompareLeftEditable)` ヘルパー関数

**`isEditable` 参照方法の混在（6ファイル）**

| ファイル | パターン |
| --- | --- |
| `DiagramBlock` | `const isEditable = editor?.isEditable ?? true` |
| `MathBlock` | `editor.isEditable` 直接参照 |
| `RegularCodeBlock` | `editor.isEditable` 直接参照 |
| `HtmlPreviewBlock` | `editor.isEditable` 直接参照 |
| `ImageNodeView` | `useBlockNodeState` から取得 |
| `MermaidNodeView` | `useEditorState` で監視 |


### 状態管理の不統一

| コンポーネント | パターン |
| --- | --- |
| `ImageNodeView`, `TableNodeView` | `useBlockNodeState` フック使用 |
| `MermaidNodeView` | 独自の `useState` + `useEditorState` |

> `MermaidNodeView` は CodeBlock の統合エントリポイントのため独自管理が必要だが、`isEditable`/`isCompareLeft` の取得方法は統一すべき。


### 複雑な条件式（3+ 真偽値）

| ファイル | 行 | 条件数 |
| --- | --- | --- |
| `DiagramBlock.tsx` | 137 | 5条件 |
| `MathBlock.tsx` | 66 | 4条件 |
| `RegularCodeBlock.tsx` | 51 | 3条件 |

> 抽出候補: `shouldShowBorder(isSelected, isCompareLeft, isCompareLeftEditable, isEditable, editOpen)` ヘルパー関数


### Props 肥大化

| コンポーネント | Props数 |
| --- | --- |
| `EditorMainContent` | 29+ |
| `EditorMenuPopovers` | 22 |
| `InlineMergeView` | 13 |

> `EditorMenuPopovers` の Anchor 状態（6ペア）をオブジェクトに集約可能。


### 分割候補

| ファイル | 行数 | 分割案 |
| --- | --- | --- |
| `diffHighlight.ts` | 517 | フラット diff / セマンティック diff / Decoration を分離 |
| `diffEngine.ts` | 451 | semantic diff を `semanticDiffEngine.ts` に分離 |
| `InlineMergeView.tsx` | 496 | ファイル I/O / エディタ同期 / レイアウトを分離 |
| `useEditorFileOps.ts` | 372 | ファイル操作 / PDF / マージを分離 |


## vscode-extension


### ゴッドファイル

| ファイル | 行数 |
| --- | --- |
| `providers/SpecDocsProvider.ts` | 600 |
| `providers/ChangesProvider.ts` | 528 |
| `providers/MarkdownEditorProvider.ts` | 459 |
| `extension.ts` | 451 |


### 重複ロジック

**エラーメッセージ抽出（14箇所）**

```typescript
// 14箇所で繰り返されるパターン
const msg = e instanceof Error ? e.message : String(e);
vscode.window.showErrorMessage(`... failed: ${msg}`);
```

> 抽出候補: `showError(operation: string, e: unknown)` ユーティリティ

**ドロップ先ディレクトリ解決（4箇所）**

`SpecDocsProvider` 内の `createFile`, `createFolder`, `importFiles`, `handleExternalDrop` で同一ロジック。

> 抽出候補: `resolveDestDir(item, rootPaths)` メソッド

**TreeDataProvider ボイラープレート（6プロバイダ）**

全プロバイダが同一の `EventEmitter` + `getTreeItem` + `refresh` パターンを繰り返している。

> 抽出候補: `BaseTreeProvider<T>` 抽象クラス

**`isMarkdownFile` 判定（5箇所）**

> 抽出候補: 共有ユーティリティ `shared/fileUtils.ts`


### 型安全性

| 問題 | ファイル | 行 |
| --- | --- | --- |
| `as unknown as string` | `extension.ts` | 202, 206 |
| `as unknown as` | `App.tsx` | 240 |
| インラインメッセージ型 | `MarkdownEditorProvider.ts` | 215 |

> メッセージ型を `interface WebviewMessage` として定義すべき。


### サイレントエラー（空 catch）

8箇所で `catch { /* ignore */ }` が使用されている。\
Git コマンド失敗時にログを出力すべき。


### アーキテクチャ

`MarkdownEditorProvider` が God Object 化（11の責務）。

- Custom Editor Provider
- Webview ライフサイクル
- メッセージルーティング
- ファイル監視
- 外部変更検知
- テーマ同期
- 設定同期
- Undo/Redo 追跡
- デバウンス管理

> 分割候補: `EditorProvider`, `WebviewManager`, `MessageRouter`


## web-app


### ゴッドファイル

| ファイル | 行数 |
| --- | --- |
| `components/ExplorerPanel.tsx` | 1708 |
| `app/docs/edit/CardAreaPanel.tsx` | 490 |
| `app/docs/edit/useLayoutEditor.ts` | 334 |
| `app/api/github/content/route.ts` | 301 |


### ExplorerPanel の肥大化（1708行）

**26 個の useState** を持つ巨大コンポーネント。\
ファイルエクスプローラ、ブランチ管理、コミット履歴、ドラッグ＆ドロップ、リネーム、ファイル作成、認証を 1 ファイルに集約。

> 分割候補:
>
> - `ExplorerTreeView` — ファイルツリー表示
> - `ExplorerBranchPanel` — ブランチ管理
> - `ExplorerCommitList` — コミット履歴
> - `useExplorerState` — 状態管理フック


### ページコンポーネントの複雑性

`app/markdown/page.tsx`（293行）に 12 個の `useState` + 3 `useRef` + 7 `useEffect`。

> 抽出候補: `useEditorPageState` カスタムフック


### Fetch パターンの重複

GitHub Content API の呼び出しが 3 箇所以上で重複。\
パスエンコーディングも 4 箇所で同一ロジック。

> 抽出候補: `lib/githubApi.ts` に集約


## 改善ロードマップ


### Quick Win（1-2 日）

| # | 改善 | 対象 |
| --- | --- | --- |
| 1 | 比較モード条件をヘルパー関数に抽出 | editor-core（4ファイル） |
| 2 | `showBorder` 条件をヘルパー関数に抽出 | editor-core（4ファイル） |
| 3 | エラーメッセージユーティリティ抽出 | vscode-extension（14箇所） |
| 4 | `resolveDestDir` メソッド抽出 | vscode-extension（4箇所） |


### 短期（1-2 週間）

| # | 改善 | 対象 |
| --- | --- | --- |
| 5 | `diffHighlight.ts` 分割 | editor-core |
| 6 | `diffEngine.ts` からセマンティック diff 分離 | editor-core |
| 7 | `MarkdownEditorProvider` 責務分離 | vscode-extension |
| 8 | GitHub API ユーティリティ集約 | web-app |
| 9 | `isEditable` 取得方法の統一 | editor-core |


### 中期（1-2 ヶ月）

| # | 改善 | 対象 |
| --- | --- | --- |
| 10 | `ExplorerPanel` 分割（1708行 → 4コンポーネント） | web-app |
| 11 | `InlineMergeView` 分割 | editor-core |
| 12 | `BaseTreeProvider` 抽象クラス導入 | vscode-extension |
| 13 | `EditorMenuPopovers` props 集約 | editor-core |
| 14 | `useEditorFileOps` 責務分離 | editor-core |


## 使用スキル

- `code-review-checklist`
