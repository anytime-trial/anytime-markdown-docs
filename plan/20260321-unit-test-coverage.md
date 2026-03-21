# ユニットテストカバレッジ拡充計画

作成日: 2026-03-21

## 背景

テストカバレッジ監査の結果、純粋関数を含むユーティリティ7ファイル、エクステンション内ロジック2箇所、スタイル関数7ファイルにテストがないことが判明した。
フック25ファイルは DOM/エディタ依存が強いため、純粋ロジック部分のみを対象とする。


## 実装順序

### Phase 1: 純粋関数ユーティリティ（優先度高）

1. `mermaidConfig.ts` — `extractMermaidConfig`, `mergeMermaidConfig`
2. `plantumlConfig.ts` — `extractPlantUmlConfig`, `mergePlantUmlConfig`
3. `commentHelpers.ts` — `parseCommentData`, `preprocessComments`, `appendCommentData`
4. `blockClipboard.ts` — `findBlockNode`, `BLOCK_NODE_TYPES` 等（エディタ依存部分は除外）
5. `markdownSerializer.ts` — `embedImageAnnotations`, `embedGifSettings`（エディタ依存）
6. `editorContentLoader.ts` — `setTrailingNewline`, `getTrailingNewline` の追加テスト
7. `fileHandleStore.ts` — IndexedDB モック使用

### Phase 2: エクステンション内純粋ロジック

8. `diffHighlight.ts` — `computeBlockDiff()`
9. `changeGutterExtension.ts` — `getChangedPositions()`

### Phase 3: スタイル関数

10. `editorStyles.ts` — `getEditorPaperSx()`
11. `headingStyles.ts` — `getHeadingStyles()`
12. `codeStyles.ts` — `getCodeStyles()`
13. `baseStyles.ts` — `getBaseStyles()`
14. `blockStyles.ts` — `getBlockStyles()`
15. `inlineStyles.ts` — `getInlineStyles()`
16. `printStyles.ts` — `PrintStyles`


## 方針

- TDD: テストを先に書き、fail を確認してから実装（今回は既存コードのテスト追加なので、テスト作成→pass 確認の流れ）
- 純粋関数のみ対象。DOM/エディタ依存のフックは対象外
- エディタインスタンスが必要な関数は `createTestEditor` を使用
- モックは最小限に抑える
