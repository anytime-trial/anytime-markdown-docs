# カラー一覧

更新日: 2026-03-21


## 1. 色定義の方針

- エディタ UI の色定数は `packages/editor-core/src/constants/colors.ts` に集約する。
- ダーク/ライトモードの切替はヘルパー関数（`getTextPrimary(isDark)` 等）で行う。
- シンタックスハイライトは `styles/codeStyles.ts`、見出しスタイルは `styles/headingStyles.ts` に定義する。
- CSS カスタムプロパティ（`--vscode-*`）は `globals.css` で VS Code テーマ互換の値を提供する。
- ランディングページ固有の色はコンポーネント内に直接定義する。


## 2. エディタ色定数（`colors.ts`）

### 2.1 エディタ背景色

| 定数名 | Dark | Light | 備考 |
| --- | --- | --- | --- |
| `DEFAULT_DARK_BG` / `DEFAULT_LIGHT_BG` | `#0D1117` | `#F8F9FA` | エディタ本文背景。ユーザー設定で上書き可 |
| `DEFAULT_DARK_CODE_BG` / `DEFAULT_LIGHT_CODE_BG` | `#161B22` | `#F1F5F9` | コードブロック背景 |
| `DEFAULT_DARK_HEADING_BG` / `DEFAULT_LIGHT_HEADING_BG` | `#1A202C` | `#EDF2F7` | 見出しセクション背景 |
| `CAPTURE_BG` | - | `#ffffff` | 画像エクスポート Canvas 背景 |


### 2.2 エディタ文字色

| 定数名 | Dark | Light | 備考 |
| --- | --- | --- | --- |
| `DEFAULT_DARK_TEXT` / `DEFAULT_LIGHT_TEXT` | `#E2E8F0` | `#2D3748` | エディタ本文文字色。ユーザー設定で上書き可 |
| `DEFAULT_DARK_HEADING_LINK` / `DEFAULT_LIGHT_HEADING_LINK` | `#63B3ED` | `#3182CE` | 見出しリンク色 |


### 2.3 UI テキスト色（MUI テーマ準拠）

| 定数名 | Dark | Light |
| --- | --- | --- |
| `TEXT_PRIMARY` | `#ffffffde` | `#000000de` |
| `TEXT_SECONDARY` | `#ffffff99` | `#00000099` |
| `TEXT_DISABLED` | `#ffffff61` | `#00000061` |


### 2.4 UI 背景色（MUI テーマ準拠）

| 定数名 | Dark | Light |
| --- | --- | --- |
| `BG_PAPER` | `#121212` | `#fff` |
| `ACTION_HOVER` | `rgba(255,255,255,0.08)` | `rgba(0,0,0,0.04)` |
| `ACTION_SELECTED` | `rgba(255,255,255,0.16)` | `rgba(0,0,0,0.08)` |
| `DIVIDER` | `rgba(255,255,255,0.12)` | `rgba(0,0,0,0.12)` |


### 2.5 UI アクセント色（MUI テーマ準拠）

| カテゴリ | Dark | Light |
| --- | --- | --- |
| Primary Main | `#90caf9` | `#1976d2` |
| Primary Dark | `#42a5f5` | `#1565c0` |
| Primary Light | `#e3f2fd` | `#42a5f5` |
| Primary Contrast | `rgba(0,0,0,0.87)` | `#fff` |
| Error Main | `#f44336` | `#d32f2f` |
| Warning Main | `#ffa726` | `#ed6c02` |
| Warning Light | `#ffb74d` | `#ff9800` |
| Success Main | `#66bb6a` | `#2e7d32` |

Grey:

| シェード | 値 |
| --- | --- |
| 100 | `#f5f5f5` |
| 300 | `#e0e0e0` |
| 900 | `#212121` |


### 2.6 アクセントカラー

| 定数名 | 値 | 用途 |
| --- | --- | --- |
| `ACCENT_COLOR` | `#e8a012` | 検索ハイライト（現在のマッチ） |
| `ACCENT_COLOR_ALPHA` | `rgba(232,160,18,0.35)` | 検索ハイライト（他のマッチ） |


### 2.7 Admonition（GitHub 準拠）

| タイプ | 色 |
| --- | --- |
| NOTE | `#1f6feb`（青） |
| TIP | `#238636`（緑） |
| IMPORTANT | `#8957e5`（紫） |
| WARNING | `#d29922`（黄） |
| CAUTION | `#da3633`（赤） |


### 2.8 PlantUML

| 定数名 | 値 | 用途 |
| --- | --- | --- |
| `PLANTUML_DARK_FG` | `#CCCCCC` | 前景色 |
| `PLANTUML_DARK_BG` | `#2D2D2D` | 背景色 |
| `PLANTUML_DARK_SURFACE` | `#1E1E1E` | サーフェス色 |


### 2.9 その他

| 定数名 | 値 | 用途 |
| --- | --- | --- |
| `FILE_DROP_OVERLAY_COLOR` | `rgba(66,133,244,0.15)` | ファイル D&D オーバーレイ |
| `COMMON_WHITE` | `#fff` | 汎用白色 |


## 3. ソースモード Base64 バッジ色

`components/SourceModeEditor.tsx` に定義。

| 定数名 | Dark | Light |
| --- | --- | --- |
| `BASE64_BADGE` | `rgba(139,92,246,0.25)` | `rgba(139,92,246,0.18)` |
| `BASE64_BORDER` | `rgba(139,92,246,0.5)` | `rgba(139,92,246,0.4)` |

> Violet-500（`#8B5CF6`）ベースの半透明色。


## 4. シンタックスハイライト色

`styles/codeStyles.ts` に定義。lowlight（highlight.js）のテーマ色。

| 要素 | Dark | Light | 説明 |
| --- | --- | --- | --- |
| keyword / selector-tag / built_in / type | `#ff7b72` | `#cf222e` | キーワード（赤） |
| string / attr | `#a5d6ff` | `#0a3069` | 文字列 |
| comment / doctag | `#8b949e` | `#6e7781` | コメント（灰） |
| number / literal / variable / regexp | `#79c0ff` | `#0550ae` | 数値（青） |
| title / class / function | `#d2a8ff` | `#8250df` | 関数名（紫） |
| params | `#c9d1d9` | `#24292f` | パラメータ |
| meta / symbol / bullet | `#ffa657` | `#953800` | メタ |
| addition テキスト | `#aff5b4` | `#116329` | 追加行（緑） |
| addition 背景 | `rgba(46,160,67,0.15)` | `rgba(46,160,67,0.15)` | 追加行背景 |
| deletion テキスト | `#ffdcd7` | `#82071e` | 削除行（赤） |
| deletion 背景 | `rgba(248,81,73,0.15)` | `rgba(248,81,73,0.15)` | 削除行背景 |


## 5. 見出しスタイル色

`styles/headingStyles.ts` に定義。

| 要素 | Dark | Light |
| --- | --- | --- |
| H1 左ボーダー | `#63B3ED` | `#3182CE` |
| H1 背景グラデーション | `rgba(99,179,237,0.12)` | `rgba(49,130,206,0.08)` |
| H2 左ボーダー | `rgba(99,179,237,0.6)` | `rgba(49,130,206,0.5)` |
| H2 背景グラデーション | `rgba(99,179,237,0.08)` | `rgba(49,130,206,0.05)` |
| H3 左ボーダー | `rgba(99,179,237,0.35)` | `rgba(49,130,206,0.3)` |


## 6. インラインスタイル色

`styles/inlineStyles.ts` に定義。

### 6.1 コメントハイライト

| 要素 | 色 |
| --- | --- |
| 背景 | `rgba(255,200,0,0.25)` |
| ボーダー | `rgba(255,200,0,0.6)` |
| ホバー背景 | `rgba(255,200,0,0.4)` |
| ポイントマーカー | `rgba(255,200,0,0.8)` |

### 6.2 検索マッチ

| 要素 | 色 |
| --- | --- |
| マッチ背景 | `alpha(warning-light, 0.3〜0.5)` |
| 現在のマッチ背景 | `alpha(warning-main, 0.4〜0.5)` |
| 現在のマッチアウトライン | `primary-main` |


## 7. Diff/マージ色

`extensions/diffHighlight.ts` に定義。

| 要素 | 色 | 用途 |
| --- | --- | --- |
| 左ブロック（削除） | `rgba(248,81,73,0.10)` | 削除行背景 |
| 右ブロック（追加） | `rgba(46,160,67,0.10)` | 追加行背景 |
| 左セル（削除） | `rgba(248,81,73,0.18)` | 削除セル背景 |
| 右セル（追加） | `rgba(46,160,67,0.18)` | 追加セル背景 |
| ニュートラル | `rgba(128,128,128,0.06)` | 変更なし領域 |


## 8. 画像アノテーション色

`types/imageAnnotation.ts` に定義。6色パレット。

| ラベル | 色 |
| --- | --- |
| Red | `#ef4444` |
| Blue | `#3b82f6` |
| Green | `#22c55e` |
| Yellow | `#eab308` |
| White | `#ffffff` |
| Black | `#000000` |


## 9. スクロールバー色

`styles/editorStyles.ts` に定義。

| 要素 | Dark | Light |
| --- | --- | --- |
| トラック | `rgba(255,255,255,0.45)` | `rgba(0,0,0,0.4)` |
| ホバー | `rgba(255,255,255,0.6)` | `rgba(0,0,0,0.55)` |
| 外側背景 | `rgba(255,255,255,0.03)` | `rgba(0,0,0,0.04)` |


## 10. CSS カスタムプロパティ（`globals.css`）

`packages/web-app/src/app/globals.css` で定義。VS Code テーマ互換の CSS 変数。

### 10.1 エディタ基本色

| 変数 | Dark | Light |
| --- | --- | --- |
| `--vscode-editor-background` | `#1e1e1e` | `#ffffff` |
| `--vscode-editor-foreground` | `#d4d4d4` | `#1f1f1f` |
| `--vscode-panel-border` | `#2d2d2d` | `#e5e5e5` |
| `--vscode-focusBorder` | `#007fd4` | `#005a9e` |

### 10.2 ツールバー操作色

| 変数 | Dark | Light |
| --- | --- | --- |
| `--vscode-toolbar-hoverBackground` | `rgba(90,93,94,0.31)` | `rgba(0,0,0,0.06)` |
| `--vscode-toolbar-activeBackground` | `rgba(127,127,127,0.2)` | `rgba(0,0,0,0.1)` |

### 10.3 テキスト色

| 変数 | Dark | Light |
| --- | --- | --- |
| `--vscode-textBlockQuote-border` | `#007acc` | `#0078d4` |
| `--vscode-textBlockQuote-foreground` | `#cccccc` | `#57606a` |
| `--vscode-textLink-foreground` | `#3794ff` | `#0969da` |
| `--vscode-textCodeBlock-background` | `rgba(10,10,10,0.4)` | `rgba(220,220,220,0.4)` |
| `--vscode-descriptionForeground` | `#9e9e9e` | `#717171` |
| `--vscode-errorForeground` | `#f44747` | `#d32f2f` |

### 10.4 検索ハイライト色

| 変数 | Dark | Light |
| --- | --- | --- |
| `--vscode-editor-findMatchHighlightBackground` | `#ffd33d44` | `rgba(234,172,0,0.4)` |
| `--vscode-editor-findMatchBackground` | `rgba(162,113,0,0.6)` | `rgba(162,113,0,0.4)` |
| `--vscode-editor-findMatchBorder` | `#f0a000` | `#d18616` |
| `--vscode-editor-selectionBackground` | `rgba(0,120,215,0.3)` | `rgba(0,120,215,0.2)` |

### 10.5 入力コントロール色

| 変数 | Dark | Light |
| --- | --- | --- |
| `--vscode-input-border` | `#3c3c3c` | `#cecece` |
| `--vscode-input-foreground` | `#cccccc` | `#1f1f1f` |
| `--vscode-input-background` | （未定義） | `#ffffff` |
| `--vscode-input-placeholderForeground` | `#a6a6a6` | `#767676` |

### 10.6 ウィジェット色

| 変数 | Dark | Light |
| --- | --- | --- |
| `--vscode-editorWidget-background` | `#252526` | `#f3f3f3` |
| `--vscode-editorWidget-border` | `#454545` | `#c8c8c8` |
| `--vscode-button-background` | `#0078d4` | `#0078d4` |
| `--vscode-button-hoverBackground` | `#006cbd` | `#006cbd` |

### 10.7 シンタックスアイコン色

| 変数 | Dark | Light | 対象 |
| --- | --- | --- | --- |
| `symbolIcon-keywordForeground` | `#569cd6` | `#0000ff` | キーワード |
| `symbolIcon-classForeground` | `#4ec9b0` | `#267f99` | クラス |
| `symbolIcon-methodForeground` | `#dcdcaa` | `#795e26` | メソッド |
| `symbolIcon-stringForeground` | `#ce9178` | `#a31515` | 文字列 |
| `symbolIcon-numberForeground` | `#b5cea8` | `#098658` | 数値 |
| `symbolIcon-commentForeground` | `#6a9955` | `#008000` | コメント |
| `symbolIcon-variableForeground` | `#9cdcfe` | `#001080` | 変数 |
| `symbolIcon-colorForeground` | `#d16969` | `#811f3f` | カラー |
| `symbolIcon-namespaceForeground` | `#c586c0` | `#af00db` | 名前空間 |
| `gitDecoration-deletedResourceForeground` | `#f44747` | `#d32f2f` | Git 削除ファイル |


## 11. ランディングページ固有色

`packages/web-app/src/app/components/LandingBody.tsx` に定義。

### 11.1 ヒーロー / CTA

| 要素 | Dark | Light |
| --- | --- | --- |
| ヒーローグラデーション | `rgba(232,160,18,0.08)` | `rgba(232,160,18,0.06)` |
| CTA ボタンホバー | `#d4920e` | `#d4920e` |
| CTA ボタンテキスト | `#000000` | `#000000` |
| ボタンシャドウ | `rgba(232,160,18,0.25)` | `rgba(232,160,18,0.3)` |
| ボタンホバーシャドウ | `rgba(232,160,18,0.35)` | `rgba(232,160,18,0.4)` |

### 11.2 カード

| 要素 | Dark | Light |
| --- | --- | --- |
| ボーダー | `rgba(255,255,255,0.08)` | `rgba(0,0,0,0.08)` |
| 背景 | `rgba(255,255,255,0.03)` | `rgba(0,0,0,0.02)` |
| ホバーボーダー | `rgba(255,255,255,0.12)` | `rgba(0,0,0,0.12)` |

### 11.3 その他

| 要素 | Dark | Light |
| --- | --- | --- |
| 機能ラベル | `ACCENT_COLOR` | `#9a6b00` |
| シャドウ | `rgba(0,0,0,0.4)` | `rgba(0,0,0,0.12)` |


## 12. ヘルパー関数一覧

`constants/colors.ts` で定義。`isDark: boolean` を受け取り、モードに応じた色を返す。

| 関数名 | 返す色 |
| --- | --- |
| `getEditorBg` | エディタ背景色（ユーザー設定考慮） |
| `getEditorText` | エディタ文字色（ユーザー設定考慮） |
| `getEditDialogBg` | ブロック編集ダイアログ背景色 |
| `getTextPrimary` | UI テキスト Primary |
| `getTextSecondary` | UI テキスト Secondary |
| `getTextDisabled` | UI テキスト Disabled |
| `getBgPaper` | UI Paper 背景色 |
| `getActionHover` | UI ホバー色 |
| `getActionSelected` | UI 選択色 |
| `getDivider` | UI 区切り線色 |
| `getPrimaryMain` | Primary メイン色 |
| `getPrimaryDark` | Primary ダーク色 |
| `getPrimaryLight` | Primary ライト色 |
| `getPrimaryContrast` | Primary コントラスト色 |
| `getErrorMain` | Error メイン色 |
| `getWarningMain` | Warning メイン色 |
| `getWarningLight` | Warning ライト色 |
| `getSuccessMain` | Success メイン色 |
| `getGrey` | Grey シェード（100/300/900） |
