# ソースモード Base64 折りたたみ表示

作成日: 2026-03-21

## 背景

ソースモードで画像をbase64で埋め込んでいる場合、非常に長い文字列が表示され可読性が低い。
base64データを折りたたんで短いトークンで表示し、データの安全性を保ちながら可読性を改善する。

## 設計

### アプローチ

表示用テキスト置換方式。`sourceText`（state）は常に完全なbase64データを保持し、textareaに表示するテキストのみ短いトークンに置換する。

### データフロー

```
sourceText (完全なbase64含む、useSourceMode state)
  ↓ collapseBase64()
displayText + tokenMap (表示用短縮テキスト + 元データの対応表)
  ↓ textarea表示
ユーザー編集 → 新displayText
  ↓ restoreBase64(newDisplayText, tokenMap)
新sourceText → onSourceChange() → state更新
```

### トークン形式

```markdown
# 変換前
![screenshot](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAA...長大な文字列...)

# 変換後（textarea表示）
![screenshot](data:base64-image-0)
```

- `data:base64-image-N` は連番付きの一意トークン
- `data:` プレフィックスを残し、既存のdata URL判定ロジック（`src.startsWith("data:")`)に影響しない
- 短いので1行に収まり、行番号も正確

### 新規ファイル

**`packages/editor-core/src/utils/base64Collapse.ts`**

- `collapseBase64(text: string): { displayText: string; tokenMap: Map<string, string> }`
  - 正規表現で `data:image/...;base64,...` パターンを検出
  - 各マッチを `data:base64-image-N` トークンに置換
  - tokenMap に `トークン → 元のdata URL` を格納
- `restoreBase64(displayText: string, tokenMap: Map<string, string>): string`
  - displayText 内のトークンを tokenMap で元のdata URLに復元

### 変更ファイル

**`packages/editor-core/src/components/SourceModeEditor.tsx`**

- props の `sourceText` から `collapseBase64()` で displayText を導出（useMemo）
- tokenMap を useRef で保持
- textarea の `value` と mirror に displayText を使用
- `onChange` で `restoreBase64()` を通してから `onSourceChange` を呼出
- `onCopy`/`onCut` ハンドラで選択範囲内のトークンを元データに復元してクリップボードへ

### エッジケース

| ケース | 対応 |
|--------|------|
| 画像行を削除 | トークンが消える → restoreで自然に除外 → 正常 |
| 新しいbase64を貼付 | 未トークン化のまま保存 → 次回render時にトークン化 |
| トークンを部分編集 | パターン不一致 → restoreでそのまま残る → sourceTextに影響なし |
| コピー操作 | クリップボードハンドラが元データに復元 |
| 画像が0個 | collapseが何もせずパススルー → オーバーヘッドなし |

### 安全性

- `sourceText`（useSourceMode state）は常に完全なデータを保持
- トークン置換は表示レイヤーのみ
- 保存は常に復元後の完全なsourceTextを経由
- tokenMapはrefで保持し、displayText生成ごとに更新

## 実装計画

1. `base64Collapse.ts` — `collapseBase64`/`restoreBase64` ユーティリティ作成（TDD）
2. `SourceModeEditor.tsx` — displayText導出、onChange復元、mirror連動
3. `SourceModeEditor.tsx` — クリップボードハンドラ（onCopy/onCut）
4. 結合テスト・ビルド検証
