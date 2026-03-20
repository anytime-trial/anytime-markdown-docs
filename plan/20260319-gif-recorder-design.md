# GIF レコーダーブロック設計書

更新日: 2026-03-19

## 概要

ブラウザの Screen Capture API で画面をキャプチャし、指定した矩形領域の操作を録画してアニメーション GIF を生成するブロック要素。

## ユーザーフロー

1. スラッシュコマンド `/gif` で GIF ブロックを挿入
2. ブロック内の「録画」アイコンをクリック
3. ブラウザの画面共有ダイアログでウィンドウ/画面を選択
4. プレビュー上でドラッグして矩形領域を選択
5. 「録画開始」→ 操作を実行（最大30秒）
6. 「録画停止」→ GIF エンコード（プログレスバー表示）
7. プレビュー再生 → 「保存」で images/ に保存 + Markdown 挿入

## Markdown 表現

```markdown
![Basic editing demo](images/basic-editing-demo.gif)
<!-- gif-settings: {"fps": 10, "width": 800, "duration": 12.5} -->
```

- 他のビューアでは通常の GIF 画像として再生
- Anytime Markdown では専用 NodeView で再生コントロール付き表示

## GIF 専用ブロック（NodeView）

### インライン表示

- GIF 画像のプレビュー
- ツールバー: 録画ボタン / 再生・一時停止 / 速度調整（0.5x/1x/2x） / 編集（ブロック要素編集画面）

### ブロック要素編集画面（全画面ダイアログ）

- 左: GIF フレームのタイムライン（サムネイル一覧、フレーム削除可能）
- 右: プレビュー再生
- トリミング: 開始/終了フレームの指定

## 録画ダイアログ

```
┌─────────────────────────────────────────┐
│ GIF Recorder                        [×] │
├─────────────────────────────────────────┤
│  ┌───────────────────────────────────┐  │
│  │                                   │  │
│  │   画面共有プレビュー               │  │
│  │   ┌─────────────┐                │  │
│  │   │ 選択矩形    │ ← ドラッグ      │  │
│  │   └─────────────┘                │  │
│  │                                   │  │
│  └───────────────────────────────────┘  │
│                                         │
│  [矩形選択] [●録画開始]  00:00 / 00:30  │
│                                         │
│  出力幅: 800px   FPS: 10               │
└─────────────────────────────────────────┘
```

### 状態遷移

1. **初期**: 「画面を選択」ボタンのみ表示
2. **画面選択済み**: プレビュー表示 + 矩形選択モード
3. **矩形選択済み**: 「録画開始」ボタンが有効化
4. **録画中**: タイマー表示、「録画停止」ボタン、赤い枠点滅
5. **エンコード中**: プログレスバー表示
6. **完了**: GIF プレビュー + 「保存」「やり直し」ボタン

## 技術構成

| 要素 | 技術 |
|---|---|
| 画面キャプチャ | `navigator.mediaDevices.getDisplayMedia()` |
| 矩形選択 | Canvas overlay + マウスドラッグ |
| フレーム抽出 | `requestAnimationFrame` + Canvas `drawImage` (100ms 間隔 = 10FPS) |
| GIF エンコード | gif.js (Web Worker で非同期処理) |
| 保存 | VS Code: postMessage → ファイル保存 / Web: Blob download |
| TipTap 拡張 | gifExtension（Node 定義） |

## ファイル構成

```
packages/editor-core/src/
  extensions/gifExtension.ts          — TipTap Node 定義
  components/GifNodeView.tsx          — インライン表示 + ツールバー
  components/GifRecorderDialog.tsx    — 録画ダイアログ（画面共有・矩形選択・録画）
  components/GifPlayerDialog.tsx      — 編集画面（タイムライン・プレビュー・トリミング）
  utils/gifEncoder.ts                 — gif.js ラッパー
```

## 仕様詳細

### GIF 設定

- FPS: 10（固定）
- 出力幅: 800px（アスペクト比維持）
- 最大録画時間: 30 秒
- ループ: 無限

### Markdown パーサー連携

- `![...](*.gif)` の直後に `<!-- gif-settings: {...} -->` がある場合、GIF 専用 NodeView で表示
- gif-settings がない場合は通常の画像として表示

### VS Code 連携

- 録画完了後、postMessage で GIF バイナリを拡張側に送信
- 拡張側で images/ に保存し、webviewUri を返却
- 画像保存と同じ方式（saveClipboardImage と同様のフロー）

## 依存パッケージ

- `gif.js`: GIF エンコード（Web Worker 対応）

## 利用スキル

- `superpowers:brainstorming`（設計）
- `superpowers:writing-plans`（実装計画）
- `superpowers:test-driven-development`（実装）
