# テキスト色定数化 実装計画

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** MUI テーマのテキスト色参照（`"text.primary"` / `"text.secondary"` / `"text.disabled"`）を実際の色値定数に置換する

**Architecture:** `constants/colors.ts` にダーク/ライト各モードのテキスト色定数を追加し、各コンポーネントで `isDark` 判定を使って適切な定数を選択する。sx prop 内の `"text.xxx"` 文字列と、スタイルファイル内の `theme.palette.text.xxx` の両方を対象とする。

**Tech Stack:** MUI v5, React, TypeScript

---

## 現状

- MUI デフォルトのテキスト色（カスタマイズなし）:
  - Dark: `text.primary` = `#ffffffde`, `text.secondary` = `#ffffff99`, `text.disabled` = `#ffffff61`
  - Light: `text.primary` = `#000000de`, `text.secondary` = `#00000099`, `text.disabled` = `#00000061`
- 37 ファイル、99 箇所で使用
- 2 パターン: sx prop 文字列（`"text.secondary"`）とスタイル関数（`theme.palette.text.secondary`）

## 対象の分類

### パターン A: sx prop 文字列（コンポーネント内）— 78 箇所、30 ファイル

コンポーネントで `color: "text.secondary"` のように使用。置換には `isDark` が必要。
多くのコンポーネントは既に `useTheme()` または `isDark` を持つ。持たないものは追加する。

### パターン B: スタイル関数（`theme` 引数あり）— 21 箇所、7 ファイル

スタイルファイルで `theme.palette.text.secondary` として使用。`theme.palette.mode === "dark"` で分岐可能。

- `styles/blockStyles.ts` (4)
- `styles/headingStyles.ts` (2)
- `styles/baseStyles.ts` (1)
- `components/mergeTiptapStyles.ts` (3)
- `components/SourceModeEditor.tsx` (3)
- `components/FullscreenDiffView.tsx` (2)
- `components/MergeEditorPanel.tsx` (2)
- `components/OutlinePanel.tsx` (5) — `theme.palette.text` + sx 混在
- `components/LinePreviewPanel.tsx` (1)

## タスク

### Task 1: 定数を追加

**Files:**
- Modify: `packages/editor-core/src/constants/colors.ts`

**Step 1: 定数を追加**

`colors.ts` の既存テキスト色定数の下に以下を追加:

```typescript
// ── UI テキスト色（MUI テーマ準拠） ──
export const DARK_TEXT_PRIMARY = "#ffffffde";
export const DARK_TEXT_SECONDARY = "#ffffff99";
export const DARK_TEXT_DISABLED = "#ffffff61";
export const LIGHT_TEXT_PRIMARY = "#000000de";
export const LIGHT_TEXT_SECONDARY = "#00000099";
export const LIGHT_TEXT_DISABLED = "#00000061";

/** ダーク/ライトモードに応じた UI テキスト色を返すヘルパー */
export function getTextPrimary(isDark: boolean): string {
  return isDark ? DARK_TEXT_PRIMARY : LIGHT_TEXT_PRIMARY;
}
export function getTextSecondary(isDark: boolean): string {
  return isDark ? DARK_TEXT_SECONDARY : LIGHT_TEXT_SECONDARY;
}
export function getTextDisabled(isDark: boolean): string {
  return isDark ? DARK_TEXT_DISABLED : LIGHT_TEXT_DISABLED;
}
```

**Step 2: ビルド確認**

Run: `npx tsc --noEmit -p packages/editor-core/tsconfig.json`
Expected: エラーなし

**Step 3: コミット**

```
feat(editor-core): テキスト色定数を colors.ts に追加
```

---

### Task 2: スタイルファイルを置換（パターン B）

**Files:**
- Modify: `packages/editor-core/src/styles/blockStyles.ts`
- Modify: `packages/editor-core/src/styles/headingStyles.ts`
- Modify: `packages/editor-core/src/styles/baseStyles.ts`
- Modify: `packages/editor-core/src/components/mergeTiptapStyles.ts`

**Step 1: 各ファイルの `theme.palette.text.xxx` を定数に置換**

スタイル関数は `theme` を引数に取るため、`const isDark = theme.palette.mode === "dark"` を追加し、`theme.palette.text.primary` → `getTextPrimary(isDark)` に変更。

例（blockStyles.ts）:
```typescript
import { getTextPrimary, getTextSecondary } from "../constants/colors";

// 関数内:
const isDark = theme.palette.mode === "dark";
// theme.palette.text.secondary → getTextSecondary(isDark)
// theme.palette.text.primary → getTextPrimary(isDark)
```

**Step 2: ビルド確認**

Run: `npx tsc --noEmit -p packages/editor-core/tsconfig.json`

**Step 3: テスト**

Run: `npm test --workspace=packages/editor-core`

**Step 4: コミット**

```
refactor(editor-core): スタイルファイルのテキスト色を定数に置換
```

---

### Task 3: コンポーネントを置換（パターン A）— text.secondary

最も多い `"text.secondary"`（約 55 箇所、27 ファイル）を置換。

**対象ファイル一覧:**

既に `isDark` または `useTheme()` がある:
- `TableNodeView.tsx`, `ImageNodeView.tsx`, `CommentPanel.tsx`, `OutlinePanel.tsx`,
  `FrontmatterBlock.tsx`, `EditorToolbar.tsx`, `SourceModeEditor.tsx`,
  `FullscreenDiffView.tsx`, `MergeEditorPanel.tsx`, `LinePreviewPanel.tsx`,
  `SearchReplaceBar.tsx`, `SourceSearchBar.tsx`, `CodeBlockEditDialog.tsx`

`isDark` の追加が必要:
- `EditDialogHeader.tsx`, `EditorDialogs.tsx`, `EditorContextMenu.tsx`,
  `StatusBar.tsx`, `EditorSettingsPanel.tsx`, `SamplePanel.tsx`,
  `ZoomToolbar.tsx`, `ImageCropTool.tsx`, `GifRecorderDialog.tsx`,
  `GifPlayerDialog.tsx`, `HtmlPreviewBlock.tsx`, `MathBlock.tsx`,
  `BlockInlineToolbar.tsx`, `DiagramBlock.tsx`, `ImageAnnotationDialog.tsx`,
  `SlashCommandMenu.tsx`, `EditorErrorBoundary.tsx`

**Step 1: 各ファイルで置換**

- `import { getTextSecondary } from "../constants/colors"` を追加
- `isDark` が無い場合は `const isDark = useTheme().palette.mode === "dark"` を追加
- `color: "text.secondary"` → `color: getTextSecondary(isDark)` に置換
- `color="text.secondary"` (Typography prop) → `sx={{ color: getTextSecondary(isDark) }}` に変更

**Step 2: ビルド確認**

Run: `npx tsc --noEmit -p packages/editor-core/tsconfig.json`

**Step 3: テスト**

Run: `npm test --workspace=packages/editor-core`

**Step 4: コミット**

```
refactor(editor-core): text.secondary を定数に置換
```

---

### Task 4: コンポーネントを置換（パターン A）— text.primary, text.disabled

残りの `"text.primary"`（約 8 箇所）と `"text.disabled"`（約 15 箇所）を置換。

**text.primary 対象:**
- `LineNumberTextarea.tsx`, `SearchReplaceBar.tsx`, `SourceSearchBar.tsx`,
  `CodeBlockEditDialog.tsx`, `EditorToolbar.tsx`

**text.disabled 対象:**
- `ImageNodeView.tsx`, `GifNodeView.tsx`, `DiagramBlock.tsx`,
  `ImageCropTool.tsx`, `InlineMergeView.tsx`, `CommentPanel.tsx`,
  `LineNumberTextarea.tsx`

**Step 1: 各ファイルで置換**

- `getTextPrimary` / `getTextDisabled` を import に追加
- `color: "text.primary"` → `color: getTextPrimary(isDark)` に置換
- `color: "text.disabled"` → `color: getTextDisabled(isDark)` に置換

**Step 2: ビルド確認**

Run: `npx tsc --noEmit -p packages/editor-core/tsconfig.json`

**Step 3: テスト**

Run: `npm test --workspace=packages/editor-core`

**Step 4: コミット**

```
refactor(editor-core): text.primary/text.disabled を定数に置換
```

---

### Task 5: 最終検証

**Step 1: 残存確認**

```bash
grep -r '"text\.\(primary\|secondary\|disabled\)"' packages/editor-core/src/ --include='*.ts' --include='*.tsx' | grep -v __tests__ | grep -v node_modules
grep -r 'theme\.palette\.text\.\(primary\|secondary\|disabled\)' packages/editor-core/src/ --include='*.ts' --include='*.tsx' | grep -v __tests__ | grep -v node_modules
```

Expected: 0 件

**Step 2: ビルド確認**

Run: `npx tsc --noEmit -p packages/editor-core/tsconfig.json`

**Step 3: 全テスト**

Run: `npm test --workspace=packages/editor-core`

**Step 4: VS Code 拡張ビルド**

Run: `npm run compile --workspace=packages/vscode-extension`

## 注意事項

- `alpha(theme.palette.text.secondary, 0.6)` のような箇所は `alpha(getTextSecondary(isDark), 0.6)` に置換
- 三項演算子内の条件分岐（例: `isFolded ? "text.disabled" : "text.primary"`）は `isFolded ? getTextDisabled(isDark) : getTextPrimary(isDark)` に
- `color="text.secondary"` の Typography prop 形式は sx prop に変更する
- テストファイル（`__tests__/`）内の参照は変更しない
