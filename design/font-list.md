# フォント一覧

更新日: 2026-03-21


## 1. フォント読み込み戦略

- 外部フォントファイル（`.woff`, `.woff2`, `.ttf`, `.otf`）はバンドルしない。
- Web フォントは Playfair Display のみ。Next.js のフォント最適化経由で読み込む。
- その他はすべてシステムフォントスタックに依存する。
- KaTeX フォント（数式レンダリング用）は `node_modules` 内に含まれる。


## 2. フォントファミリー一覧

### 2.1 Playfair Display（Web フォント）

| 項目 | 値 |
| --- | --- |
| 種別 | Web フォント（Google Fonts） |
| ウェイト | 700 |
| 読み込み | `next/font/google`（`display: 'swap'`） |
| フォールバック | Georgia, "Times New Roman", serif |
| 用途 | ランディングページのヒーロー見出し |

定義箇所:

- `packages/web-app/src/app/components/LandingPage.tsx`

使用箇所:

- `packages/web-app/src/app/components/LandingPage.tsx` — ヒーロー見出し
- `packages/web-app/src/app/components/LandingBody.tsx` — セクション見出し


### 2.2 システム UI フォントスタック

| 項目 | 値 |
| --- | --- |
| 種別 | システムフォント |
| フォントスタック | `-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif` |
| CSS 変数 | `--vscode-font-family` |
| 用途 | アプリケーション全体の UI テキスト（ボタン、メニュー、ダイアログ等） |

定義箇所:

- `packages/web-app/src/app/globals.css`

使用箇所:

- エディタ UI 全般（ツールバー、パネル、ダイアログ）
- VS Code 拡張ランディングページ（`packages/vscode-extension/src/webview/App.tsx`）
- エラーページ（`packages/web-app/src/app/global-error.tsx`）
- OGP 画像生成（`packages/web-app/src/app/opengraph-image.tsx`）


### 2.3 コードフォントスタック（モノスペース）

| 項目 | 値 |
| --- | --- |
| 種別 | システムフォント |
| フォントスタック | `'Cascadia Code', 'Fira Code', Menlo, Monaco, 'Courier New', monospace` |
| CSS 変数 | `--vscode-editor-font-family` |
| 用途 | コードブロック、シンタックスハイライト、ソースモード |

定義箇所:

- `packages/web-app/src/app/globals.css`

フォント選択の優先順位:

| 優先度 | フォント | 提供元 |
| --- | --- | --- |
| 1 | Cascadia Code | Microsoft（Windows Terminal 同梱） |
| 2 | Fira Code | Mozilla |
| 3 | Menlo | Apple（macOS 同梱） |
| 4 | Monaco | Apple（macOS 同梱） |
| 5 | Courier New | 汎用フォールバック |
| 6 | monospace | OS デフォルト |


### 2.4 `monospace`（インライン使用）

| 項目 | 値 |
| --- | --- |
| 種別 | ジェネリックキーワード |
| 用途 | コンポーネント内のインラインモノスペース表示 |

MUI `sx` prop の `fontFamily: "monospace"` として使用。OS デフォルトのモノスペースフォントに解決される。

主な使用箇所:

- `SourceModeEditor.tsx` — ソースモードテキストエリア
- `FrontmatterBlock.tsx` — YAML 表示
- `CodeBlockEditDialog.tsx` — コードエディタ
- `SearchReplaceBar.tsx` / `SourceSearchBar.tsx` — 検索 UI
- `ImageNodeView.tsx` — 画像メタデータキャプション
- `EditorSettingsPanel.tsx` — 設定値表示
- `MergeEditorPanel.tsx` — マージビュー情報


### 2.5 `sans-serif`（エディタ本文）

| 項目 | 値 |
| --- | --- |
| 種別 | ジェネリックキーワード |
| 用途 | エディタ本文のフォールバック |

定義箇所:

- `packages/editor-core/src/styles/editorStyles.ts` — `.tiptap` エディタ本文


### 2.6 Roboto（MUI 暗黙デフォルト）

| 項目 | 値 |
| --- | --- |
| 種別 | システムフォント |
| 用途 | Material-UI コンポーネントのデフォルト |

> MUI のデフォルト `typography.fontFamily` として暗黙的に参照される。\
> アプリケーションではグローバルフォントスタック（2.2）で上書きされるため、直接使用されることは少ない。


## 3. フォントサイズ定数

`packages/editor-core/src/constants/dimensions.ts` で定義。

### UI コンポーネント

| コンポーネント | サイズ |
| --- | --- |
| ステータスバー | 0.875rem（14px） |
| ツールバー | 0.8rem（12.8px） |
| アウトラインパネル | 0.8rem（12.8px） |
| コメント本文 | 0.8rem（12.8px） |
| コンテキストメニュー | 0.8125rem（13px） |
| メニュー項目 | 0.85rem（13.6px） |
| ダイアログヘッダー | 0.875rem（14px） |
| コメント入力 | 0.875rem（14px） |
| スキップリンク | 0.875rem（14px） |

### 補助テキスト

| コンポーネント | サイズ |
| --- | --- |
| ハンドルバーキャプション | 0.65rem（10px） |
| 検索カウンター | 0.65rem（10px） |
| チップ | 0.7rem（11.2px） |
| 小ボタン / キャプション | 0.7rem（11.2px） |
| パネルボタン | 0.75rem（12px） |
| ショートカットヒント | 0.75rem（12px） |
| 全画面パネルヘッダー / タブ | 0.75rem（12px） |
| 見出しアンカー | 0.75rem（12px） |
| フロントマターコード | 0.75rem（12px） |
| マージ情報 | 0.75rem（12px） |
| 検索入力 | 0.78rem（12.5px） |
| スラッシュコマンドメニュー | 0.85rem（13.6px） |

### バッジ

| コンポーネント | サイズ |
| --- | --- |
| バッジ番号 | 0.55rem（8.8px） |
| 見出しバッジ | 0.6rem（9.6px） |
| マージバッジ | 0.6rem（9.6px） |
| ツールチップ | 12px |
