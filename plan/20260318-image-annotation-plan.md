# 画像アノテーション 実装計画

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** 画像のハンドルバーからアノテーション編集画面を開き、矩形・丸・直線を SVG で描画してコメントを記載する機能を追加する。

**Architecture:** 画像ノードの `attrs.annotations` に JSON 文字列で図形データを保存。`ImageAnnotationDialog` 全画面ダイアログで SVG オーバーレイ描画。インライン表示時は SVG オーバーレイで図形を表示。Markdown では `data-annotations` 属性で保存。

**Tech Stack:** React, MUI, SVG, TipTap (ProseMirror), TypeScript

---


### Task 1: 画像拡張に `annotations` 属性を追加

**Files:**

- Modify: `packages/editor-core/src/imageExtension.ts`

**Step 1: `annotations` 属性を追加**

```typescript
// addAttributes() の return に追加
annotations: {
  default: null,
  parseHTML: (element: HTMLElement) => element.getAttribute("data-annotations") || null,
  renderHTML: (attributes: Record<string, unknown>) => {
    if (!attributes.annotations) return {};
    return { "data-annotations": attributes.annotations };
  },
},
```

**Step 2: 型チェック**

Run: `npx tsc --noEmit`\
Expected: エラーなし

**Step 3: コミット**

```
git add packages/editor-core/src/imageExtension.ts
git commit -m "feat(editor-core): 画像ノードに annotations 属性を追加"
```

---


### Task 2: アノテーションの型定義

**Files:**

- Create: `packages/editor-core/src/types/imageAnnotation.ts`

**Step 1: 型定義を作成**

```typescript
export interface ImageAnnotation {
  id: string;
  type: "rect" | "circle" | "line";
  x1: number;  // 開始 X（画像に対する %）
  y1: number;  // 開始 Y（画像に対する %）
  x2: number;  // 終了 X（画像に対する %）
  y2: number;  // 終了 Y（画像に対する %）
  color: string;
}

export type AnnotationTool = "rect" | "circle" | "line" | "eraser";

export const ANNOTATION_COLORS = [
  { label: "Red", value: "#ef4444" },
  { label: "Blue", value: "#3b82f6" },
  { label: "Green", value: "#22c55e" },
  { label: "Yellow", value: "#eab308" },
  { label: "White", value: "#ffffff" },
  { label: "Black", value: "#000000" },
] as const;

export function parseAnnotations(json: string | null): ImageAnnotation[] {
  if (!json) return [];
  try { return JSON.parse(json); } catch { return []; }
}

export function serializeAnnotations(annotations: ImageAnnotation[]): string | null {
  if (annotations.length === 0) return null;
  return JSON.stringify(annotations);
}
```

**Step 2: 型チェック**

Run: `npx tsc --noEmit`\
Expected: エラーなし

**Step 3: コミット**

```
git add packages/editor-core/src/types/imageAnnotation.ts
git commit -m "feat(editor-core): 画像アノテーション型定義を追加"
```

---


### Task 3: SVG オーバーレイコンポーネント

**Files:**

- Create: `packages/editor-core/src/components/AnnotationOverlay.tsx`

**Step 1: SVG レンダリングコンポーネントを作成**

SVG 要素で `ImageAnnotation[]` を描画する読み取り専用コンポーネント。\
画像の上に `position: absolute` で重ねて表示。

- `rect`: `<rect>` 要素（塗りなし、枠線のみ）
- `circle`: `<ellipse>` 要素（塗りなし、枠線のみ）
- `line`: `<line>` 要素

座標は `%` ベースなので `viewBox="0 0 100 100"` + `preserveAspectRatio="none"` で描画。

**Step 2: 型チェック**

Run: `npx tsc --noEmit`\
Expected: エラーなし

**Step 3: コミット**

```
git add packages/editor-core/src/components/AnnotationOverlay.tsx
git commit -m "feat(editor-core): アノテーション SVG オーバーレイコンポーネントを追加"
```

---


### Task 4: アノテーション編集ダイアログ

**Files:**

- Create: `packages/editor-core/src/components/ImageAnnotationDialog.tsx`

**Step 1: 全画面ダイアログを作成**

構成:

- ツールバー: 矩形 / 丸 / 直線 / 消しゴム の ToggleButtonGroup + 色選択 + 閉じるボタン
- メイン: 画像 + SVG オーバーレイ（描画可能）
- SVG 上でマウスドラッグで図形を描画
- 消しゴムツール選択時は図形クリックで削除

Props:

```typescript
interface ImageAnnotationDialogProps {
  open: boolean;
  onClose: () => void;
  src: string;
  annotations: ImageAnnotation[];
  onSave: (annotations: ImageAnnotation[]) => void;
  t: (key: string) => string;
}
```

State:

```typescript
const [tool, setTool] = useState<AnnotationTool>("rect");
const [color, setColor] = useState("#ef4444");
const [items, setItems] = useState<ImageAnnotation[]>(annotations);
const [drawing, setDrawing] = useState<{ x1: number; y1: number } | null>(null);
```

マウスイベント:

- `onMouseDown`: 描画開始（座標を `%` に変換）
- `onMouseMove`: プレビュー表示（描画中の図形をリアルタイム表示）
- `onMouseUp`: 図形確定、`items` に追加
- 消しゴム: `onClick` で該当図形を `items` から削除

閉じるボタン: `onSave(items)` を呼んでダイアログを閉じる。

**Step 2: 型チェック**

Run: `npx tsc --noEmit`\
Expected: エラーなし

**Step 3: コミット**

```
git add packages/editor-core/src/components/ImageAnnotationDialog.tsx
git commit -m "feat(editor-core): 画像アノテーション編集ダイアログを追加"
```

---


### Task 5: ImageNodeView にアノテーション機能を統合

**Files:**

- Modify: `packages/editor-core/src/ImageNodeView.tsx`

**Step 1: ハンドルバーにアノテーションアイコンを追加**

`BlockInlineToolbar` の `extra` に `ChatBubbleOutline` アイコンボタンを追加。\
クリックで `ImageAnnotationDialog` を開く。

**Step 2: インライン表示時に SVG オーバーレイを表示**

画像コンテナの中に `AnnotationOverlay` を配置。\
`node.attrs.annotations` からデータを読み取って表示。

**Step 3: アノテーション保存**

`ImageAnnotationDialog` の `onSave` で `updateAttributes({ annotations: serializeAnnotations(items) })` を呼ぶ。

**Step 4: 型チェック・テスト**

Run: `npx tsc --noEmit && cd packages/editor-core && npx jest --no-coverage`\
Expected: 全通過

**Step 5: コミット**

```
git add packages/editor-core/src/ImageNodeView.tsx
git commit -m "feat(editor-core): ImageNodeView にアノテーション機能を統合"
```

---


### Task 6: i18n キーの追加

**Files:**

- Modify: `packages/editor-core/src/i18n/en.json`
- Modify: `packages/editor-core/src/i18n/ja.json`

**Step 1: キーを追加**

```json
// en.json
"annotate": "Annotate",
"annotationRect": "Rectangle",
"annotationCircle": "Circle",
"annotationLine": "Line",
"annotationEraser": "Eraser",
"annotationColor": "Color",

// ja.json
"annotate": "アノテーション",
"annotationRect": "矩形",
"annotationCircle": "丸",
"annotationLine": "直線",
"annotationEraser": "消しゴム",
"annotationColor": "色",
```

**Step 2: コミット**

```
git add packages/editor-core/src/i18n/en.json packages/editor-core/src/i18n/ja.json
git commit -m "feat(editor-core): 画像アノテーション i18n キーを追加"
```

---


### Task 7: ビルド検証・最終テスト

**Files:**

- 全パッケージ

**Step 1: editor-core テスト**

Run: `cd packages/editor-core && npx jest --no-coverage`\
Expected: 全通過

**Step 2: 型チェック**

Run: `npx tsc --noEmit`\
Expected: エラーなし

**Step 3: web-app ビルド**

Run: `cd packages/web-app && npx next build`\
Expected: ビルド成功

**Step 4: vscode-extension 型チェック**

Run: `npx tsc --noEmit`\
Expected: エラーなし

**Step 5: コミット（最終調整がある場合）**

```
git add -A
git commit -m "feat(editor-core): 画像アノテーション最終検証"
```
