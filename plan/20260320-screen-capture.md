# Screen Capture（Webアプリ専用）実装計画

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Webアプリで Screen Capture API を使い、画面キャプチャ → トリミング → 画像挿入を実現する

**Architecture:** 既存の `GifRecorderDialog` の画面共有パターンを流用し、単一フレームのPNGキャプチャに特化した `ScreenCaptureDialog` を新規作成。キャプチャ後は既存の `ImageCropTool` でトリミングし、base64画像としてエディタに挿入する。VS Code環境では非表示。

**Tech Stack:** Screen Capture API (`getDisplayMedia`)、Canvas API、React、MUI

---

## Task 1: ScreenCaptureDialog コンポーネント作成

**Files:**
- Create: `packages/editor-core/src/components/ScreenCaptureDialog.tsx`

**Step 1: コンポーネント実装**

`GifRecorderDialog` のスクリーン共有ロジックを参考に、以下の状態遷移で実装:
- `idle` → `previewing`（画面共有選択中/プレビュー表示中）→ `captured`（キャプチャ完了、トリミングUI表示）

```typescript
interface ScreenCaptureDialogProps {
  open: boolean;
  onClose: () => void;
  onCapture: (dataUrl: string) => void;
  t: (key: string) => string;
}
```

処理フロー:
1. ダイアログオープン時に `getDisplayMedia()` を呼び出し
2. 画面共有のプレビューを `<video>` で表示
3. 「キャプチャ」ボタンクリックで現在のフレームを Canvas に描画 → DataURL 取得
4. 画面共有ストリームを停止
5. `ImageCropTool` でトリミングUI表示
6. トリミング確定で `onCapture(dataUrl)` コールバック呼び出し

**Step 2: 型チェック確認**

Run: `npx tsc --noEmit -p packages/editor-core/tsconfig.json`

**Step 3: コミット**

```
feat(editor-core): スクリーンキャプチャダイアログを追加
```

---

## Task 2: 翻訳キー追加

**Files:**
- Modify: `packages/editor-core/src/i18n/ja.json`
- Modify: `packages/editor-core/src/i18n/en.json`

**Step 1: 翻訳キー追加**

```json
// ja.json
"screenCapture": "スクリーンキャプチャ",
"screenCaptureStart": "画面を選択",
"screenCaptureShoot": "キャプチャ",
"screenCaptureRetry": "やり直す",

// en.json
"screenCapture": "Screen Capture",
"screenCaptureStart": "Select screen",
"screenCaptureShoot": "Capture",
"screenCaptureRetry": "Retry",
```

**Step 2: コミット**

```
feat(editor-core): スクリーンキャプチャの翻訳キーを追加
```

---

## Task 3: フッター挿入メニューにボタン追加

**Files:**
- Modify: `packages/editor-core/src/components/EditorFooterOverlays.tsx`
- Modify: `packages/editor-core/src/components/EditorMenuPopovers.tsx`（必要に応じて）

**Step 1: 挿入メニューにスクリーンキャプチャボタンを追加**

- 既存のダイアグラム挿入ボタンの近くに配置
- `window.__vscode` が存在する場合は非表示
- クリックで `ScreenCaptureDialog` を開く
- `onCapture` コールバックで `editor.chain().focus().setImage({ src: dataUrl, alt: "" }).run()`

**Step 2: 型チェック確認**

Run: `npx tsc --noEmit -p packages/editor-core/tsconfig.json`

**Step 3: コミット**

```
feat(editor-core): フッターにスクリーンキャプチャボタンを追加
```

---

## Task 4: スラッシュコマンド追加

**Files:**
- Modify: `packages/editor-core/src/extensions/slashCommandItems.ts`

**Step 1: `/screenshot` スラッシュコマンド追加**

- 既存の `/image` コマンドを参考に `/screenshot` を追加
- `window.__vscode` が存在する場合はコマンド一覧から除外
- `getDisplayMedia()` の存在チェック（`navigator.mediaDevices?.getDisplayMedia`）
- 実行時は `ScreenCaptureDialog` を開くイベントを発火

**Step 2: 型チェック確認**

Run: `npx tsc --noEmit -p packages/editor-core/tsconfig.json`

**Step 3: コミット**

```
feat(editor-core): /screenshot スラッシュコマンドを追加
```

---

## Task 5: 結合テスト・動作確認

**Step 1: ビルド確認**

Run: `npx tsc --noEmit -p packages/editor-core/tsconfig.json`

**Step 2: 既存テストが壊れていないことを確認**

Run: `npm test --workspace=packages/editor-core`

**Step 3: 手動動作確認項目**

- [ ] Webアプリでスクリーンキャプチャボタンが表示される
- [ ] クリックで画面共有ダイアログが表示される
- [ ] 画面選択後にプレビューが表示される
- [ ] キャプチャボタンでスクリーンショットが取得される
- [ ] トリミング後に画像がエディタに挿入される
- [ ] VS Code拡張ではボタンが非表示であること
- [ ] `getDisplayMedia` 未対応ブラウザで適切にフォールバック

**Step 4: コミット**

```
test(editor-core): スクリーンキャプチャの動作確認完了
```

---

## 注意事項

- `getDisplayMedia()` はHTTPS環境が必要（localhost は例外）
- モバイルブラウザでは未対応のため、機能検出で非表示にする
- キャプチャ画像はPNG base64 として挿入。サイズが大きい場合はCanvas で圧縮を検討
- 画面共有ストリームは必ず停止する（メモリリーク防止）
- エラーハンドリング: ユーザーが画面共有を拒否した場合のフォールバック
