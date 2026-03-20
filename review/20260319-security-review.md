# セキュリティレビュー

更新日: 2026-03-19

対象: develop ブランチの全変更（79 コミット）


## サマリー

| 重大度 | 件数 | 修正済み |
| --- | --- | --- |
| Critical | 1 | 1 |
| High | 3 | 3 |
| Medium | 3 | 1 |
| Low | 3 | 1 |


## Critical

### 1. `overwriteImage` のパストラバーサル

**ファイル**: `packages/vscode-extension/src/providers/MarkdownEditorProvider.ts`\
**修正コミット**: `cf94571`

**問題**: `overwriteImage` メッセージハンドラで、ウェブビューから受け取ったパスを `path.resolve()` で絶対パスに解決するが、ドキュメントディレクトリ内であることを検証していなかった。\
`../../.vscode/settings.json` のようなパスで任意のファイルを上書き可能。

**修正**:

```typescript
if (!absPath.startsWith(docDir + path.sep) && absPath !== docDir) {
  vscode.window.showErrorMessage('Path traversal detected');
  break;
}
```


## High

### 2. `saveClipboardImage` のファイル名未検証

**ファイル**: `packages/vscode-extension/src/providers/MarkdownEditorProvider.ts`\
**修正コミット**: `cf94571`

**問題**: ウェブビューから受け取った `fileName` にパス区切り文字が含まれる場合、`path.join(imagesDir, fileName)` で images ディレクトリ外に書き込み可能。

**修正**:

- ファイル名に `/`, `\`, `.` 開始を含む場合は拒否
- 解決パスが images ディレクトリ内であることを検証

### 3. メッセージプロパティの型検証不足

**ファイル**: `packages/vscode-extension/src/providers/MarkdownEditorProvider.ts`\
**修正コミット**: `cf94571`

**問題**: `message as { type: string; dataUrl: string; ... }` の TypeScript 型アサーションは実行時チェックを行わない。\
不正なメッセージで `undefined` プロパティへのアクセスが発生する。

**修正**: `typeof message.dataUrl === 'string'` の実行時型ガードに変更。

### 4. `<base href>` の CSP ギャップ

**ファイル**: `packages/vscode-extension/src/providers/MarkdownEditorProvider.ts`, `packages/vscode-extension/src/webview/App.tsx`\
**修正コミット**: `cf94571`

**問題**: CSP に `base-uri` ディレクティブがなく、`<base href>` に任意の URI を設定可能。\
`javascript:` スキームの注入でスクリプト実行のリスク。

**修正**:

- CSP に `base-uri ${webview.cspSource}` を追加
- `setBaseUri` メッセージの URI を `vscode-webview-resource:` / `https:` スキームのみに制限


## Medium

### 5. Canvas taint エラー未処理

**ファイル**: `packages/editor-core/src/components/ImageCropTool.tsx`\
**修正コミット**: `cf94571`

**問題**: CORS 制限のある画像ソースで `canvas.toDataURL()` が tainted canvas エラーをスローする。\
`crossOrigin="anonymous"` は設定済みだが、サーバーが CORS ヘッダーを返さない場合にエラーが未処理。

**修正**: `try-catch` で taint エラーをキャッチし、`console.warn` でログ出力。

### 6. `LinkValidationProvider` の ReDoS リスク

**ファイル**: `packages/vscode-extension/src/providers/LinkValidationProvider.ts`\
**状態**: 許容リスク

**問題**: コードフェンス検出の正規表現 `/^```` [\s\S]*?^```` /gm` が、大量の未閉じフェンスでバックトラッキングする可能性。

**緩和策**: ファイル保存時のみ実行されるため、インタラクティブな操作には影響しない。\
将来的に `sanitizeMarkdown.ts` と同じ文字列スキャン方式への移行を推奨。

### 7. `sanitizeMarkdown` の正規表現バックトラッキング

**ファイル**: `packages/editor-core/src/utils/sanitizeMarkdown.ts`\
**状態**: 緩和済み

**問題**: blockquote / span の HTML パターンで `(?:<(?!\/tag>)[^<]*)*` が深いネストで指数的バックトラッキングの可能性。

**緩和策**: DOMPurify でサニタイズ済みのコンテンツに対してのみ実行されるため、攻撃的な入力は到達しない。


## Low

### 8. CSV 数式インジェクション

**ファイル**: `packages/editor-core/src/utils/boxTableToMarkdown.ts`\
**状態**: 低リスク

**問題**: 罫線テーブル変換時にセル内容をサニタイズしない。\
`=SUM(A1:A10)` が Markdown テーブルに含まれ、CSV エクスポート時に Excel で実行される可能性。

**緩和策**: Markdown テーブル自体は数式を実行しない。下流ツールの問題。

### 9. `openLink` のエンコーディングバイパス

**ファイル**: `packages/vscode-extension/src/providers/MarkdownEditorProvider.ts`\
**状態**: 緩和済み

**問題**: `decodeURIComponent` でダブルエンコーディングのリスク。\
既存の `path.isAbsolute()` + `path.resolve()` チェックで十分に緩和。

### 10. メッセージ型アサーション

**ファイル**: `packages/vscode-extension/src/providers/MarkdownEditorProvider.ts`\
**修正コミット**: `cf94571`（Critical/High の修正に含む）

**問題**: `as` キャストを `typeof` ガードに置換。


## 良好な実装

| 観点 | 状態 | 備考 |
| --- | --- | --- |
| XSS 防止 | 良好 | DOMPurify でサニタイズ済み（HtmlPreviewBlock, DiagramBlock） |
| SVG インジェクション | 良好 | `SVG_SANITIZE_CONFIG` で制限 |
| CSP | 良好 | nonce ベースの script-src、base-uri 追加済み |
| クリップボード API | 良好 | try-catch + フォールバック |
| 画像アノテーション | 良好 | 構造化データ（JSON）で保存、HTML 注入なし |
| コメント機能 | 良好 | Plugin State で管理、Markdown コメントブロックで保存 |
| ファイルシステムアクセス | 修正済み | パストラバーサル防止チェック追加 |


## 使用スキル

- `code-review-checklist`（セキュリティ観点）
