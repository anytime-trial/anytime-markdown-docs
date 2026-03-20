# GIF レコーダーブロック実装計画

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** ブラウザの Screen Capture API で画面の矩形領域を録画し、アニメーション GIF として保存するブロック要素を追加する。

**Architecture:** TipTap の Node 拡張として `gifBlock` を定義し、ReactNodeViewRenderer で GifNodeView を描画する。録画は `getDisplayMedia` + Canvas でフレーム抽出し、gif.js でエンコードする。Markdown 上は `![alt](path.gif)` + `<!-- gif-settings: {...} -->` コメントで表現。

**Tech Stack:** TipTap Node extension, React, Canvas API, getDisplayMedia, gif.js, Material-UI

---

## Task 1: gif.js パッケージ追加・型定義

**Files:**
- Modify: `packages/editor-core/package.json`
- Create: `packages/editor-core/src/types/gif.js.d.ts`

**Step 1: gif.js をインストール**

```bash
cd packages/editor-core
npm install gif.js@0.2.0 --save-exact
```

**Step 2: 型定義を作成**

gif.js の Worker ファイルは node_modules から直接参照する。`@types/gif.js` は古いため自前で定義。

```typescript
// packages/editor-core/src/types/gif.js.d.ts
declare module "gif.js" {
  interface GIFOptions {
    workers?: number;
    quality?: number;
    width?: number;
    height?: number;
    workerScript?: string;
    repeat?: number;
    background?: string;
    transparent?: string | null;
    dither?: boolean | string;
  }

  interface AddFrameOptions {
    delay?: number;
    copy?: boolean;
    dispose?: number;
  }

  class GIF {
    constructor(options?: GIFOptions);
    addFrame(
      element: CanvasRenderingContext2D | HTMLCanvasElement | HTMLImageElement | ImageData,
      options?: AddFrameOptions,
    ): void;
    on(event: "finished", callback: (blob: Blob) => void): this;
    on(event: "progress", callback: (progress: number) => void): this;
    on(event: "abort", callback: () => void): this;
    render(): void;
    abort(): void;
    running: boolean;
  }

  export default GIF;
}
```

**Step 3: コミット**

```bash
git add packages/editor-core/package.json package-lock.json packages/editor-core/src/types/gif.js.d.ts
git commit -m "feat(editor-core): gif.js パッケージと型定義を追加"
```

---

## Task 2: gifEncoder ユーティリティ（TDD）

**Files:**
- Create: `packages/editor-core/src/utils/gifEncoder.ts`
- Test: `packages/editor-core/src/__tests__/gifEncoder.test.ts`

**Step 1: テスト作成**

```typescript
// packages/editor-core/src/__tests__/gifEncoder.test.ts
import { extractFramesFromCanvas, GifRecorderState } from "../utils/gifEncoder";

describe("gifEncoder", () => {
  describe("GifRecorderState", () => {
    it("should initialize with idle state", () => {
      const state = new GifRecorderState();
      expect(state.status).toBe("idle");
      expect(state.frames).toHaveLength(0);
      expect(state.elapsed).toBe(0);
    });

    it("should track frames", () => {
      const state = new GifRecorderState();
      const canvas = document.createElement("canvas");
      canvas.width = 100;
      canvas.height = 100;
      state.addFrame(canvas);
      expect(state.frames).toHaveLength(1);
    });

    it("should enforce max duration", () => {
      const state = new GifRecorderState({ maxDuration: 30000, fps: 10 });
      expect(state.maxFrames).toBe(300);
    });

    it("should clear frames on reset", () => {
      const state = new GifRecorderState();
      const canvas = document.createElement("canvas");
      canvas.width = 100;
      canvas.height = 100;
      state.addFrame(canvas);
      state.reset();
      expect(state.frames).toHaveLength(0);
      expect(state.status).toBe("idle");
    });
  });

  describe("extractFramesFromCanvas", () => {
    it("should crop rectangle from source canvas", () => {
      const source = document.createElement("canvas");
      source.width = 200;
      source.height = 200;
      const ctx = source.getContext("2d")!;
      ctx.fillStyle = "red";
      ctx.fillRect(0, 0, 200, 200);

      const rect = { x: 50, y: 50, width: 100, height: 100 };
      const result = extractFramesFromCanvas(source, rect, 80);
      expect(result.width).toBe(80);
      expect(result.height).toBe(40);
    });
  });
});
```

**Step 2: テスト実行 → FAIL**

```bash
cd packages/editor-core && npx jest --testPathPattern=gifEncoder -v
```

**Step 3: 実装**

```typescript
// packages/editor-core/src/utils/gifEncoder.ts

export interface CropRect {
  x: number;
  y: number;
  width: number;
  height: number;
}

export interface GifSettings {
  fps: number;
  width: number;
  duration: number;
}

export interface GifRecorderOptions {
  maxDuration?: number; // ms, default 30000
  fps?: number;         // default 10
  outputWidth?: number; // default 800
}

export class GifRecorderState {
  status: "idle" | "selecting" | "recording" | "encoding" | "done" = "idle";
  frames: ImageData[] = [];
  elapsed = 0;
  readonly fps: number;
  readonly maxDuration: number;
  readonly outputWidth: number;
  readonly maxFrames: number;

  constructor(options?: GifRecorderOptions) {
    this.fps = options?.fps ?? 10;
    this.maxDuration = options?.maxDuration ?? 30000;
    this.outputWidth = options?.outputWidth ?? 800;
    this.maxFrames = Math.floor(this.maxDuration / 1000 * this.fps);
  }

  addFrame(canvas: HTMLCanvasElement): boolean {
    if (this.frames.length >= this.maxFrames) return false;
    const ctx = canvas.getContext("2d");
    if (!ctx) return false;
    this.frames.push(ctx.getImageData(0, 0, canvas.width, canvas.height));
    this.elapsed = this.frames.length / this.fps * 1000;
    return true;
  }

  reset(): void {
    this.frames = [];
    this.elapsed = 0;
    this.status = "idle";
  }
}

/**
 * ソース Canvas から矩形領域を切り出し、指定幅にリサイズした Canvas を返す。
 */
export function extractFramesFromCanvas(
  source: HTMLCanvasElement,
  rect: CropRect,
  targetWidth: number,
): HTMLCanvasElement {
  const scale = targetWidth / rect.width;
  const targetHeight = Math.round(rect.height * scale);
  const canvas = document.createElement("canvas");
  canvas.width = targetWidth;
  canvas.height = targetHeight;
  const ctx = canvas.getContext("2d")!;
  ctx.drawImage(source, rect.x, rect.y, rect.width, rect.height, 0, 0, targetWidth, targetHeight);
  return canvas;
}

/**
 * ImageData 配列から GIF Blob を生成する。
 * gif.js を動的インポートし Web Worker でエンコード。
 */
export async function encodeGif(
  frames: ImageData[],
  width: number,
  height: number,
  fps: number,
  onProgress?: (p: number) => void,
): Promise<Blob> {
  const GIF = (await import("gif.js")).default;
  const delay = Math.round(1000 / fps);

  return new Promise((resolve, reject) => {
    try {
      const gif = new GIF({
        workers: 2,
        quality: 10,
        width,
        height,
        workerScript: new URL("gif.js/dist/gif.worker.js", import.meta.url).href,
      });

      for (const frame of frames) {
        gif.addFrame(frame, { delay, copy: true });
      }

      gif.on("finished", (blob: Blob) => resolve(blob));
      gif.on("progress", (p: number) => onProgress?.(p));
      gif.render();
    } catch (err) {
      reject(err);
    }
  });
}
```

**Step 4: テスト実行 → PASS**

```bash
npx jest --testPathPattern=gifEncoder -v
```

**Step 5: コミット**

```bash
git add packages/editor-core/src/utils/gifEncoder.ts packages/editor-core/src/__tests__/gifEncoder.test.ts
git commit -m "feat(editor-core): GIF エンコーダーユーティリティを追加（TDD）"
```

---

## Task 3: gifExtension（TipTap Node 定義）

**Files:**
- Create: `packages/editor-core/src/extensions/gifExtension.ts`

**Step 1: Node 定義**

```typescript
// packages/editor-core/src/extensions/gifExtension.ts
import { Node, mergeAttributes } from "@tiptap/core";
import { ReactNodeViewRenderer } from "@tiptap/react";
import { GifNodeView } from "../components/GifNodeView";

export const GifBlock = Node.create({
  name: "gifBlock",
  group: "block",
  draggable: true,
  atom: true,

  addAttributes() {
    return {
      src: { default: null },
      alt: { default: "" },
      width: { default: null },
      gifSettings: {
        default: null,
        parseHTML: (element: HTMLElement) => element.getAttribute("data-gif-settings") || null,
        renderHTML: (attributes: Record<string, unknown>) => {
          if (!attributes.gifSettings) return {};
          return { "data-gif-settings": attributes.gifSettings };
        },
      },
    };
  },

  parseHTML() {
    return [
      {
        tag: 'img[src$=".gif"]',
        getAttrs: (element: HTMLElement) => {
          const src = element.getAttribute("src");
          if (!src?.endsWith(".gif")) return false;
          return {
            src,
            alt: element.getAttribute("alt") || "",
            width: element.getAttribute("width") || null,
          };
        },
      },
    ];
  },

  renderHTML({ HTMLAttributes }) {
    return ["img", mergeAttributes(HTMLAttributes, { src: HTMLAttributes.src })];
  },

  addNodeView() {
    return ReactNodeViewRenderer(GifNodeView);
  },
});
```

**Step 2: コミット**

```bash
git add packages/editor-core/src/extensions/gifExtension.ts
git commit -m "feat(editor-core): GIF ブロック TipTap Node 定義を追加"
```

---

## Task 4: GifNodeView（インライン表示）

**Files:**
- Create: `packages/editor-core/src/components/GifNodeView.tsx`

**Step 1: NodeView 実装**

```typescript
// packages/editor-core/src/components/GifNodeView.tsx
import React, { useRef, useState, useCallback, useEffect } from "react";
import { NodeViewProps, NodeViewWrapper } from "@tiptap/react";
import { Box, IconButton, Tooltip, Typography, Slider } from "@mui/material";
import FiberManualRecordIcon from "@mui/icons-material/FiberManualRecord";
import PlayArrowIcon from "@mui/icons-material/PlayArrow";
import PauseIcon from "@mui/icons-material/Pause";
import EditIcon from "@mui/icons-material/Edit";
import SpeedIcon from "@mui/icons-material/Speed";
import { useTranslations } from "../contexts/I18nContext";
import { BlockInlineToolbar } from "./codeblock/BlockInlineToolbar";
import { useBlockNodeState } from "../hooks/useBlockNodeState";
import { GifRecorderDialog } from "./GifRecorderDialog";
import { GifPlayerDialog } from "./GifPlayerDialog";

const SPEED_OPTIONS = [0.5, 1, 2];

export function GifNodeView({ editor, node, updateAttributes, getPos }: NodeViewProps) {
  const t = useTranslations("MarkdownEditor");
  const { isEditable, isSelected, handleDeleteBlock, showToolbar } = useBlockNodeState(editor, node, getPos);

  const { src, alt, gifSettings: settingsJson } = node.attrs;
  const settings = settingsJson ? JSON.parse(settingsJson as string) : null;

  const imgRef = useRef<HTMLImageElement>(null);
  const [recorderOpen, setRecorderOpen] = useState(false);
  const [playerOpen, setPlayerOpen] = useState(false);
  const [playing, setPlaying] = useState(true);
  const [speed, setSpeed] = useState(1);

  // GIF の再生/停止は canvas に描画して制御
  // 簡易版: img 要素の表示/非表示で制御

  const handleRecordComplete = useCallback(
    (blob: Blob, fileName: string, newSettings: { fps: number; width: number; duration: number }) => {
      // VS Code 環境: postMessage で保存
      const w = (window as Record<string, unknown>).__vscode as { postMessage: (msg: unknown) => void } | undefined;
      if (w) {
        const reader = new FileReader();
        reader.onload = () => {
          w.postMessage({
            type: "saveClipboardImage",
            dataUrl: reader.result,
            fileName,
          });
        };
        reader.readAsDataURL(blob);
      } else {
        // Web 環境: download
        const url = URL.createObjectURL(blob);
        const a = document.createElement("a");
        a.href = url;
        a.download = fileName;
        a.click();
        URL.revokeObjectURL(url);
        updateAttributes({ src: fileName, gifSettings: JSON.stringify(newSettings) });
      }
      setRecorderOpen(false);
    },
    [updateAttributes],
  );

  // VS Code から imageSaved メッセージを受け取る
  useEffect(() => {
    const handler = (e: MessageEvent) => {
      const msg = e.data;
      if (msg?.type === "imageSaved" && typeof msg.path === "string" && msg.path.endsWith(".gif")) {
        updateAttributes({ src: msg.webviewUri || msg.path });
      }
    };
    window.addEventListener("message", handler);
    return () => window.removeEventListener("message", handler);
  }, [updateAttributes]);

  return (
    <NodeViewWrapper data-drag-handle>
      <Box sx={{ position: "relative", display: "inline-block", width: "100%" }}>
        {src ? (
          <Box sx={{ position: "relative" }}>
            <img
              ref={imgRef}
              src={src as string}
              alt={alt as string}
              style={{ maxWidth: "100%", display: "block", borderRadius: 4 }}
            />
            {/* 再生コントロール */}
            <Box
              sx={{
                position: "absolute",
                bottom: 8,
                left: 8,
                display: "flex",
                alignItems: "center",
                gap: 0.5,
                bgcolor: "rgba(0,0,0,0.6)",
                borderRadius: 1,
                px: 1,
                py: 0.25,
              }}
            >
              <IconButton size="small" onClick={() => setPlaying(!playing)} sx={{ color: "white", p: 0.25 }}>
                {playing ? <PauseIcon fontSize="small" /> : <PlayArrowIcon fontSize="small" />}
              </IconButton>
              <Typography variant="caption" sx={{ color: "white", fontSize: "0.65rem" }}>
                {speed}x
              </Typography>
              {settings && (
                <Typography variant="caption" sx={{ color: "rgba(255,255,255,0.7)", fontSize: "0.6rem", ml: 0.5 }}>
                  {(settings.duration / 1000).toFixed(1)}s
                </Typography>
              )}
            </Box>
          </Box>
        ) : (
          <Box
            sx={{
              border: "2px dashed",
              borderColor: "divider",
              borderRadius: 1,
              p: 4,
              textAlign: "center",
              cursor: isEditable ? "pointer" : "default",
            }}
            onClick={() => isEditable && setRecorderOpen(true)}
          >
            <FiberManualRecordIcon sx={{ fontSize: 40, color: "error.main", mb: 1 }} />
            <Typography variant="body2" color="text.secondary">
              {t("gifRecordPlaceholder") ?? "Click to record GIF"}
            </Typography>
          </Box>
        )}

        {showToolbar && (
          <BlockInlineToolbar
            label="GIF"
            onEdit={() => (src ? setPlayerOpen(true) : setRecorderOpen(true))}
            onDelete={handleDeleteBlock}
            t={t}
            extra={
              isEditable ? (
                <Tooltip title={t("gifRecord") ?? "Record"}>
                  <IconButton size="small" onClick={() => setRecorderOpen(true)}>
                    <FiberManualRecordIcon sx={{ fontSize: 16, color: "error.main" }} />
                  </IconButton>
                </Tooltip>
              ) : undefined
            }
          />
        )}

        {recorderOpen && (
          <GifRecorderDialog
            open={recorderOpen}
            onClose={() => setRecorderOpen(false)}
            onComplete={handleRecordComplete}
          />
        )}

        {playerOpen && src && (
          <GifPlayerDialog
            open={playerOpen}
            onClose={() => setPlayerOpen(false)}
            src={src as string}
            settings={settings}
          />
        )}
      </Box>
    </NodeViewWrapper>
  );
}
```

**Step 2: コミット**

```bash
git add packages/editor-core/src/components/GifNodeView.tsx
git commit -m "feat(editor-core): GIF NodeView インライン表示を追加"
```

---

## Task 5: GifRecorderDialog（録画ダイアログ）

**Files:**
- Create: `packages/editor-core/src/components/GifRecorderDialog.tsx`

**Step 1: 録画ダイアログ実装**

主要ロジック:
1. `getDisplayMedia()` で画面共有ストリーム取得
2. `<video>` にストリームを表示
3. Canvas オーバーレイでドラッグ矩形選択
4. 録画中: `setInterval(100ms)` で video → Canvas → 矩形クロップ → ImageData を蓄積
5. 停止: gif.js でエンコード
6. プレビュー表示 + 保存ボタン

状態遷移: `idle` → `previewing`（画面共有中）→ `selecting`（矩形ドラッグ中）→ `ready`（矩形確定）→ `recording` → `encoding` → `done`

ダイアログは `EditDialogWrapper` + `EditDialogHeader` を使用。

このファイルは約200行。主なコンポーネント構成:
- 画面共有プレビュー（video + canvas overlay）
- 矩形選択の mousedown/mousemove/mouseup ハンドラ
- 録画タイマー（00:00 / 00:30）
- エンコードプログレスバー
- 完了後の GIF プレビュー + ファイル名入力 + 保存ボタン

**Step 2: コミット**

```bash
git add packages/editor-core/src/components/GifRecorderDialog.tsx
git commit -m "feat(editor-core): GIF 録画ダイアログを追加"
```

---

## Task 6: GifPlayerDialog（編集・再生ダイアログ）

**Files:**
- Create: `packages/editor-core/src/components/GifPlayerDialog.tsx`

**Step 1: プレイヤーダイアログ実装**

主要機能:
- GIF をフレーム分解して Canvas で再生（速度調整対応）
- フレームタイムライン（サムネイル一覧）
- フレーム削除
- 開始/終了フレームのトリミングスライダー

`EditDialogWrapper` + `EditDialogHeader` を使用。

**Step 2: コミット**

```bash
git add packages/editor-core/src/components/GifPlayerDialog.tsx
git commit -m "feat(editor-core): GIF プレイヤー/編集ダイアログを追加"
```

---

## Task 7: Markdown パーサー・シリアライザー連携

**Files:**
- Modify: `packages/editor-core/src/utils/frontmatterHelpers.ts`
- Modify: `packages/editor-core/src/utils/markdownSerializer.ts`

**Step 1: gif-settings パース**

`frontmatterHelpers.ts` の `preprocessMarkdown` に gif-settings コメントの抽出を追加。
`<!-- gif-settings: {"fps": 10, ...} -->` を画像ノードの属性に変換。

**Step 2: gif-settings シリアライズ**

`markdownSerializer.ts` に `embedGifSettings` 関数を追加。
gifBlock ノードの gifSettings 属性を `<!-- gif-settings: {...} -->` コメントとして画像の直後に出力。

**Step 3: テスト追加**

既存の roundTrip テストに GIF ブロックのケースを追加。

**Step 4: コミット**

```bash
git add packages/editor-core/src/utils/frontmatterHelpers.ts packages/editor-core/src/utils/markdownSerializer.ts
git commit -m "feat(editor-core): GIF 設定の Markdown パース/シリアライズを追加"
```

---

## Task 8: エディタ統合（スラッシュコマンド・拡張登録）

**Files:**
- Modify: `packages/editor-core/src/extensions/slashCommandItems.ts`
- Modify: `packages/editor-core/src/editorExtensions.ts`（拡張登録）
- Modify: `packages/editor-core/src/i18n/ja.json`（i18n キー追加）
- Modify: `packages/editor-core/src/i18n/en.json`（存在する場合）

**Step 1: スラッシュコマンド追加**

```typescript
{
  id: "gif",
  labelKey: "slashGif",
  icon: React.createElement(GifIcon, { fontSize: "small" }),
  keywords: ["gif", "record", "screen", "capture", "録画", "キャプチャ", "アニメーション"],
  action: (editor) => {
    editor.chain().focus().insertContent({ type: "gifBlock" }).run();
  },
},
```

**Step 2: gifExtension を拡張リストに追加**

`editorExtensions.ts` の拡張配列に `GifBlock` を追加。

**Step 3: i18n キー追加**

```json
{
  "slashGif": "GIF 録画",
  "gifRecordPlaceholder": "クリックして GIF を録画",
  "gifRecord": "録画",
  "gifRecording": "録画中...",
  "gifEncoding": "GIF 生成中...",
  "gifSelectArea": "領域を選択",
  "gifStartRecord": "録画開始",
  "gifStopRecord": "録画停止",
  "gifSave": "保存",
  "gifRetry": "やり直し",
  "gifFileName": "ファイル名",
  "gifDuration": "録画時間",
  "gifFrames": "フレーム数"
}
```

**Step 4: コミット**

```bash
git add packages/editor-core/src/extensions/slashCommandItems.ts packages/editor-core/src/editorExtensions.ts packages/editor-core/src/i18n/ja.json
git commit -m "feat(editor-core): GIF ブロックをスラッシュコマンド・エディタ拡張に統合"
```

---

## Task 9: VS Code 拡張連携

**Files:**
- Modify: `packages/vscode-extension/src/providers/MarkdownEditorProvider.ts`

**Step 1: GIF 保存ハンドラ**

既存の `saveClipboardImage` と同じフローで `.gif` ファイルを保存。
`saveClipboardImage` は既に画像全般に対応しているため、追加変更は不要の可能性が高い。
ただし `data:image/gif;base64,...` の match パターンが `/^data:image\/\w+;base64,(.+)$/` なので GIF も対応済み。

確認のみ。変更不要であればスキップ。

**Step 2: コミット（必要な場合のみ）**

---

## Task 10: ビルド・型チェック・テスト

**Step 1: 型チェック**

```bash
npx tsc --noEmit
```

**Step 2: ユニットテスト**

```bash
npm test --workspaces --if-present
```

**Step 3: webpack ビルド**

```bash
cd packages/vscode-extension && npx webpack --mode production
```

**Step 4: e2e テスト**

```bash
cd packages/web-app && npm run e2e
```

**Step 5: 修正があればコミット**

---

## 検証チェックリスト

- [ ] `/gif` スラッシュコマンドで GIF ブロックが挿入される
- [ ] プレースホルダークリックで録画ダイアログが開く
- [ ] 画面共有ダイアログでウィンドウを選択できる
- [ ] プレビュー上でドラッグして矩形を選択できる
- [ ] 録画開始/停止が動作する（最大30秒で自動停止）
- [ ] GIF エンコードが完了しプレビューが表示される
- [ ] 保存で images/ にファイルが作成され Markdown に挿入される
- [ ] 他のビューア（GitHub等）で通常の GIF として再生される
- [ ] インライン再生コントロール（再生/停止/速度）が動作する
- [ ] 編集ダイアログでフレームタイムラインが表示される
- [ ] ソースモード切替で gif-settings が保持される
- [ ] tsc / テスト / ビルド全通過
