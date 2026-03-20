# 用紙サイズ表示機能 実装計画

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** 編集・レビューモードでエディタ本文の横幅を用紙サイズに合わせた `max-width` で制限し、印刷イメージに近い表示を実現する。

**Architecture:** `EditorSettings` に `paperSize` と `paperMargin` を追加し、`editorStyles.ts` で `.tiptap` に `max-width` + 中央寄せを適用する。\
用紙幅の計算式は `(用紙横幅mm - 余白mm × 2) × (96 / 25.4)` で px に変換。\
設定パネルに Select（用紙サイズ）と Slider（余白）を追加する。

**Tech Stack:** React, MUI, TypeScript, Tiptap, localStorage

---


## Task 1: 用紙サイズ定数の定義

**Files:**

- Modify: `packages/editor-core/src/constants/dimensions.ts`

**Step 1: 用紙サイズ定数を追加**

ファイル末尾に以下を追加:

```typescript
// ── 用紙サイズ (mm) ──
export type PaperSize = "off" | "A3" | "A4" | "B4" | "B5";

export const PAPER_WIDTHS_MM: Record<Exclude<PaperSize, "off">, number> = {
  A3: 297,
  A4: 210,
  B4: 257,
  B5: 182,
};

/** 用紙サイズ選択肢の表示順 */
export const PAPER_SIZE_OPTIONS: PaperSize[] = ["off", "A3", "A4", "B4", "B5"];

/** 余白デフォルト (mm) */
export const PAPER_MARGIN_DEFAULT = 20;
export const PAPER_MARGIN_MIN = 10;
export const PAPER_MARGIN_MAX = 40;
export const PAPER_MARGIN_STEP = 5;

/** 用紙の本文幅を px で計算する */
export function calcPaperContentWidth(paperSize: Exclude<PaperSize, "off">, marginMm: number): number {
  const contentMm = PAPER_WIDTHS_MM[paperSize] - marginMm * 2;
  return Math.round(contentMm * (96 / 25.4));
}
```

**Step 2: コミット**

```bash
git add packages/editor-core/src/constants/dimensions.ts
git commit -m "feat(editor-core): 用紙サイズ定数と本文幅計算関数を追加"
```


## Task 2: EditorSettings 型にプロパティ追加

**Files:**

- Modify: `packages/editor-core/src/useEditorSettings.ts`

**Step 1: 型とデフォルト値を更新**

`EditorSettings` インターフェースに追加:

```typescript
paperSize: "off" | "A3" | "A4" | "B4" | "B5";
paperMargin: number; // mm単位、10-40
```

`DEFAULT_SETTINGS` に追加:

```typescript
paperSize: "A4",
paperMargin: 20,
```

`SETTINGS_VERSION` を `6` → `7` に変更。\
コメントを `用紙サイズ表示を追加` に更新。

**Step 2: コミット**

```bash
git add packages/editor-core/src/useEditorSettings.ts
git commit -m "feat(editor-core): EditorSettings に paperSize/paperMargin を追加"
```


## Task 3: editorStyles に max-width 適用

**Files:**

- Modify: `packages/editor-core/src/styles/editorStyles.ts`

**Step 1: import を追加**

```typescript
import { calcPaperContentWidth } from "../constants/dimensions";
import type { PaperSize } from "../constants/dimensions";
```

**Step 2: `.tiptap` スタイルに max-width を追加**

`getEditorPaperSx` 内の `"& .tiptap"` オブジェクトに以下を追加:

```typescript
// 用紙サイズ制限
...(settings.paperSize !== "off" && {
  maxWidth: calcPaperContentWidth(settings.paperSize as Exclude<PaperSize, "off">, settings.paperMargin),
  mx: "auto",
  wordBreak: "break-word",
  overflowWrap: "break-word",
}),
```

> `mx: "auto"` は MUI sx の `marginLeft: "auto", marginRight: "auto"` のショートハンド。

**Step 3: コミット**

```bash
git add packages/editor-core/src/styles/editorStyles.ts
git commit -m "feat(editor-core): 用紙サイズに応じた max-width を .tiptap に適用"
```


## Task 4: 翻訳キーの追加

**Files:**

- Modify: `packages/editor-core/src/i18n/ja.json`
- Modify: `packages/editor-core/src/i18n/en.json`

**Step 1: 日本語の翻訳キーを追加**

`MarkdownEditor` オブジェクト内に追加:

```json
"settingPaperSize": "用紙サイズ",
"settingPaperSizeOff": "OFF",
"settingPaperMargin": "余白",
```

**Step 2: 英語の翻訳キーを追加**

```json
"settingPaperSize": "Paper Size",
"settingPaperSizeOff": "OFF",
"settingPaperMargin": "Margin",
```

**Step 3: コミット**

```bash
git add packages/editor-core/src/i18n/ja.json packages/editor-core/src/i18n/en.json
git commit -m "feat(editor-core): 用紙サイズ・余白の翻訳キーを追加"
```


## Task 5: 設定パネル UI の追加

**Files:**

- Modify: `packages/editor-core/src/components/EditorSettingsPanel.tsx`

**Step 1: import を追加**

```typescript
import { FormControl, MenuItem, Select } from "@mui/material";
import { PAPER_MARGIN_MAX, PAPER_MARGIN_MIN, PAPER_MARGIN_STEP, PAPER_SIZE_OPTIONS } from "../constants/dimensions";
```

**Step 2: Spell Check セクションの前に用紙サイズ UI を追加**

`{/* Spell Check */}` コメントの直前に以下を挿入:

```tsx
{/* Paper Size */}
<Box sx={{ mb: 3 }}>
  <Typography variant="caption" sx={{ fontWeight: 600, color: "text.secondary", mb: 0.5, display: "block" }}>
    {t("settingPaperSize")}
  </Typography>
  <FormControl size="small" fullWidth>
    <Select
      value={settings.paperSize}
      onChange={(e) => updateSettings({ paperSize: e.target.value as EditorSettings["paperSize"] })}
      aria-label={t("settingPaperSize")}
    >
      {PAPER_SIZE_OPTIONS.map((size) => (
        <MenuItem key={size} value={size}>
          {size === "off" ? t("settingPaperSizeOff") : size}
        </MenuItem>
      ))}
    </Select>
  </FormControl>
</Box>

{/* Paper Margin */}
{settings.paperSize !== "off" && (
  <Box sx={{ mb: 3 }}>
    <Typography variant="caption" sx={{ fontWeight: 600, color: "text.secondary" }}>
      {t("settingPaperMargin")}
    </Typography>
    <Box sx={{ display: "flex", alignItems: "center", gap: 1, mt: 0.5 }}>
      <Slider
        value={settings.paperMargin}
        onChange={(_, v) => updateSettings({ paperMargin: v as number })}
        min={PAPER_MARGIN_MIN}
        max={PAPER_MARGIN_MAX}
        step={PAPER_MARGIN_STEP}
        size="small"
        aria-label={t("settingPaperMargin")}
        aria-valuetext={`${settings.paperMargin}mm`}
      />
      <Typography variant="body2" sx={{ minWidth: 48, textAlign: "right", fontFamily: "monospace" }}>
        {settings.paperMargin}mm
      </Typography>
    </Box>
  </Box>
)}

<Divider sx={{ mb: 2 }} />
```

**Step 3: コミット**

```bash
git add packages/editor-core/src/components/EditorSettingsPanel.tsx
git commit -m "feat(editor-core): 設定パネルに用紙サイズ・余白の UI を追加"
```


## Task 6: ユニットテスト

**Files:**

- Create: `packages/editor-core/src/__tests__/calcPaperContentWidth.test.ts`

**Step 1: テストを作成**

```typescript
import { calcPaperContentWidth } from "../constants/dimensions";

describe("calcPaperContentWidth", () => {
  it("A4 余白20mm → 643px", () => {
    expect(calcPaperContentWidth("A4", 20)).toBe(Math.round(170 * (96 / 25.4)));
  });

  it("A3 余白20mm → 972px", () => {
    expect(calcPaperContentWidth("A3", 20)).toBe(Math.round(257 * (96 / 25.4)));
  });

  it("B4 余白20mm → 820px", () => {
    expect(calcPaperContentWidth("B4", 20)).toBe(Math.round(217 * (96 / 25.4)));
  });

  it("B5 余白20mm → 537px", () => {
    expect(calcPaperContentWidth("B5", 20)).toBe(Math.round(142 * (96 / 25.4)));
  });

  it("余白を変更すると幅が変わる", () => {
    const w10 = calcPaperContentWidth("A4", 10);
    const w40 = calcPaperContentWidth("A4", 40);
    expect(w10).toBeGreaterThan(w40);
  });
});
```

**Step 2: テスト実行**

```bash
cd packages/editor-core && npx jest src/__tests__/calcPaperContentWidth.test.ts -v
```

Expected: 全5テスト PASS

**Step 3: コミット**

```bash
git add packages/editor-core/src/__tests__/calcPaperContentWidth.test.ts
git commit -m "test(editor-core): calcPaperContentWidth のユニットテストを追加"
```


## Task 7: ビルド検証

**Step 1: 型チェック**

```bash
npx tsc --noEmit
```

Expected: エラーなし

**Step 2: テスト全体実行**

```bash
cd packages/editor-core && npm test
```

Expected: 全テスト PASS

**Step 3: ビルド**

```bash
cd packages/web-app && npx next build
```

Expected: ビルド成功
