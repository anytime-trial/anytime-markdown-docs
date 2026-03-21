# 背景色定数化 実装計画

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** MUI テーマの背景色参照（`"background.paper"` / `"action.hover"` / `"divider"` 等）を実際の色値定数に置換する

**Architecture:** `constants/colors.ts` にダーク/ライト各モードの背景色定数を追加し、各コンポーネントで `isDark` 判定を使って適切な定数を選択する。テキスト色定数化（前タスク）と同じパターン。

**Tech Stack:** MUI v5, React, TypeScript

---

## MUI デフォルト値

| テーマ色 | Dark | Light |
|---|---|---|
| `background.paper` | `#121212` | `#fff` |
| `background.default` | `#121212` | `#fff` |
| `action.hover` | `rgba(255,255,255,0.08)` | `rgba(0,0,0,0.04)` |
| `action.selected` | `rgba(255,255,255,0.16)` | `rgba(0,0,0,0.08)` |
| `divider` | `rgba(255,255,255,0.12)` | `rgba(0,0,0,0.12)` |

## 対象: 36ファイル、91箇所

- `"divider"` / `borderColor: "divider"` — 51箇所
- `"action.hover"` / `"action.selected"` — 18箇所
- `"background.paper"` / `"background.default"` — 5箇所
- `theme.palette.xxx` （スタイル関数内）— 20箇所

## タスク

### Task 1: 定数を追加

**Files:** `packages/editor-core/src/constants/colors.ts`, `packages/editor-core/src/index.ts`

定数とヘルパーを追加:

```typescript
// ── UI 背景色（MUI テーマ準拠） ──
export const DARK_BG_PAPER = "#121212";
export const LIGHT_BG_PAPER = "#fff";
export const DARK_ACTION_HOVER = "rgba(255,255,255,0.08)";
export const LIGHT_ACTION_HOVER = "rgba(0,0,0,0.04)";
export const DARK_ACTION_SELECTED = "rgba(255,255,255,0.16)";
export const LIGHT_ACTION_SELECTED = "rgba(0,0,0,0.08)";
export const DARK_DIVIDER = "rgba(255,255,255,0.12)";
export const LIGHT_DIVIDER = "rgba(0,0,0,0.12)";

export function getBgPaper(isDark: boolean): string {
  return isDark ? DARK_BG_PAPER : LIGHT_BG_PAPER;
}
export function getActionHover(isDark: boolean): string {
  return isDark ? DARK_ACTION_HOVER : LIGHT_ACTION_HOVER;
}
export function getActionSelected(isDark: boolean): string {
  return isDark ? DARK_ACTION_SELECTED : LIGHT_ACTION_SELECTED;
}
export function getDivider(isDark: boolean): string {
  return isDark ? DARK_DIVIDER : LIGHT_DIVIDER;
}
```

確認: tsc, テスト

---

### Task 2: スタイルファイルの theme.palette 参照を置換（20箇所、6ファイル）

**Files:**
- `styles/blockStyles.ts` (6)
- `styles/headingStyles.ts` (2)
- `styles/codeStyles.ts` (1)
- `components/mergeTiptapStyles.ts` (9)
- `components/OutlinePanel.tsx` (1)
- `components/MergeEditorPanel.tsx` (1)

`theme.palette.divider` → `getDivider(isDark)`
`theme.palette.background.paper` → `getBgPaper(isDark)`
`theme.palette.action.hover` → `getActionHover(isDark)`
`theme.palette.action.selected` → `getActionSelected(isDark)`

確認: tsc, テスト

---

### Task 3: コンポーネントの "divider" を置換（51箇所、33ファイル）

`borderColor: "divider"` → `borderColor: getDivider(isDark)`
`bgcolor: "divider"` → `bgcolor: getDivider(isDark)`
`border: 1, borderColor: "divider"` のパターンに注意

確認: tsc, テスト

---

### Task 4: コンポーネントの "action.hover"/"action.selected"/"background.*" を置換（23箇所）

`bgcolor: "action.hover"` → `bgcolor: getActionHover(isDark)`
`bgcolor: "action.selected"` → `bgcolor: getActionSelected(isDark)`
`bgcolor: "background.paper"` → `bgcolor: getBgPaper(isDark)`
`bgcolor: "background.default"` → `bgcolor: getBgDefault(isDark)`（background.default は background.paper と同値なので getBgPaper で代用可）

確認: tsc, テスト

---

### Task 5: 最終検証

- grep で残存確認（0件）
- tsc --noEmit
- npm test
- VS Code 拡張ビルド
