# セマンティック比較 実装計画

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Markdown テキストを見出し単位でセクション分割し、見出し LCS マッチング後にセクション単位 diff を実行する `computeSemanticDiff` を追加する。

**Architecture:** `diffEngine.ts` にセクション分割・LCS マッチング・再帰 diff の純粋関数を追加。`useMergeDiff` に `semantic` オプションを追加し、`InlineMergeView` にトグル UI を配置。既存の `DiffResult` 型は変更しない。

**Tech Stack:** TypeScript, npm `diff` ライブラリ（既存依存）, React (MUI), TipTap

---


### Task 1: セクション分割の純粋関数（TDD）

**Files:**

- Create: `packages/editor-core/src/utils/sectionParser.ts`
- Create: `packages/editor-core/src/__tests__/sectionParser.test.ts`

**Step 1: テストを作成**

```typescript
// sectionParser.test.ts
import { parseMarkdownSections, type MarkdownSection } from "../utils/sectionParser";

describe("parseMarkdownSections", () => {
  test("見出しなしテキストはルートセクション1つ", () => {
    const result = parseMarkdownSections("line1\nline2");
    expect(result).toHaveLength(1);
    expect(result[0].heading).toBeNull();
    expect(result[0].bodyLines).toEqual(["line1", "line2"]);
    expect(result[0].children).toEqual([]);
  });

  test("H2 で分割される", () => {
    const text = "intro\n## A\nbody a\n## B\nbody b";
    const result = parseMarkdownSections(text);
    expect(result).toHaveLength(3); // root + A + B
    expect(result[0].heading).toBeNull();
    expect(result[0].bodyLines).toEqual(["intro"]);
    expect(result[1].heading).toBe("A");
    expect(result[1].level).toBe(2);
    expect(result[1].headingLine).toBe("## A");
    expect(result[1].bodyLines).toEqual(["body a"]);
    expect(result[2].heading).toBe("B");
  });

  test("サブ見出しが children に格納される", () => {
    const text = "## Parent\nbody\n### Child1\nchild body\n### Child2\nchild2 body";
    const result = parseMarkdownSections(text);
    expect(result).toHaveLength(1);
    expect(result[0].heading).toBe("Parent");
    expect(result[0].children).toHaveLength(2);
    expect(result[0].children[0].heading).toBe("Child1");
    expect(result[0].children[1].heading).toBe("Child2");
  });

  test("空テキストは空配列", () => {
    expect(parseMarkdownSections("")).toEqual([]);
  });

  test("コードブロック内の # は見出しとして扱わない", () => {
    const text = "## Real\n```\n## Not a heading\n```\nafter";
    const result = parseMarkdownSections(text);
    expect(result).toHaveLength(1);
    expect(result[0].heading).toBe("Real");
    expect(result[0].bodyLines).toContain("```");
  });
});
```

**Step 2: テスト実行して失敗を確認**

Run: `cd packages/editor-core && npx jest sectionParser --no-coverage`\
Expected: FAIL（モジュールが存在しない）

**Step 3: 実装**

```typescript
// sectionParser.ts
const HEADING_RE = /^(#{1,5})\s+(.+)$/;

export interface MarkdownSection {
  heading: string | null;
  level: number;
  headingLine: string;
  bodyLines: string[];
  children: MarkdownSection[];
}

export function parseMarkdownSections(text: string): MarkdownSection[] {
  if (text === "") return [];
  const lines = text.split("\n");
  return buildSections(lines, 0, lines.length, 1);
}

function buildSections(
  lines: string[], start: number, end: number, minLevel: number,
): MarkdownSection[] {
  const sections: MarkdownSection[] = [];
  let i = start;
  let inCodeBlock = false;

  // ルートセクション（最初の見出しより前の行）
  const rootBodyLines: string[] = [];
  while (i < end) {
    const line = lines[i];
    if (line.startsWith("```")) inCodeBlock = !inCodeBlock;
    if (!inCodeBlock) {
      const m = line.match(HEADING_RE);
      if (m && m[1].length >= minLevel) break;
    }
    rootBodyLines.push(line);
    i++;
  }
  if (rootBodyLines.length > 0 || i === end) {
    sections.push({
      heading: null, level: 0, headingLine: "",
      bodyLines: rootBodyLines, children: [],
    });
  }
  if (i >= end) return sections;

  // 見出しごとにセクション分割
  while (i < end) {
    const line = lines[i];
    const m = line.match(HEADING_RE);
    if (!m) { i++; continue; }
    const level = m[1].length;
    if (level < minLevel) break; // 親レベルの見出しに到達したら終了

    const headingLine = line;
    const heading = m[2].trim();
    i++;

    // このセクションの body を収集（同レベル以上の見出しまで）
    const bodyStart = i;
    inCodeBlock = false;
    while (i < end) {
      const l = lines[i];
      if (l.startsWith("```")) inCodeBlock = !inCodeBlock;
      if (!inCodeBlock) {
        const hm = l.match(HEADING_RE);
        if (hm && hm[1].length <= level) break;
      }
      i++;
    }

    // サブ見出しを再帰的に解析
    const sectionBodyLines: string[] = [];
    const children = buildChildren(lines, bodyStart, i, level + 1, sectionBodyLines);

    sections.push({
      heading, level, headingLine,
      bodyLines: sectionBodyLines, children,
    });
  }

  return sections;
}

function buildChildren(
  lines: string[], start: number, end: number, childLevel: number,
  bodyLines: string[],
): MarkdownSection[] {
  // bodyLines にはサブ見出しより前の行を格納
  let i = start;
  let inCodeBlock = false;
  while (i < end) {
    const line = lines[i];
    if (line.startsWith("```")) inCodeBlock = !inCodeBlock;
    if (!inCodeBlock) {
      const m = line.match(HEADING_RE);
      if (m && m[1].length >= childLevel) break;
    }
    bodyLines.push(line);
    i++;
  }
  if (i >= end) return [];
  return buildSections(lines, i, end, childLevel);
}
```

**Step 4: テスト実行して通過を確認**

Run: `cd packages/editor-core && npx jest sectionParser --no-coverage`\
Expected: PASS

**Step 5: コミット**

```
git add packages/editor-core/src/utils/sectionParser.ts packages/editor-core/src/__tests__/sectionParser.test.ts
git commit -m "feat(editor-core): セクション分割パーサー追加（TDD）"
```

---


### Task 2: セクション LCS マッチング（TDD）

**Files:**

- Modify: `packages/editor-core/src/utils/sectionParser.ts`
- Modify: `packages/editor-core/src/__tests__/sectionParser.test.ts`

**Step 1: テストを追加**

```typescript
import { matchSections, type SectionMatch } from "../utils/sectionParser";

describe("matchSections", () => {
  const sec = (heading: string): MarkdownSection => ({
    heading, level: 2, headingLine: `## ${heading}`,
    bodyLines: [], children: [],
  });

  test("同一見出しがマッチする", () => {
    const left = [sec("A"), sec("B")];
    const right = [sec("A"), sec("B")];
    const result = matchSections(left, right);
    expect(result).toEqual([
      { type: "matched", left: left[0], right: right[0] },
      { type: "matched", left: left[1], right: right[1] },
    ]);
  });

  test("左にのみ存在するセクション", () => {
    const left = [sec("A"), sec("B")];
    const right = [sec("A")];
    const result = matchSections(left, right);
    expect(result[0].type).toBe("matched");
    expect(result[1]).toEqual({ type: "left-only", left: left[1], right: null });
  });

  test("右にのみ存在するセクション", () => {
    const left = [sec("A")];
    const right = [sec("A"), sec("C")];
    const result = matchSections(left, right);
    expect(result[1]).toEqual({ type: "right-only", left: null, right: right[1] });
  });

  test("順序が変わった場合は LCS でマッチ", () => {
    const left = [sec("A"), sec("B"), sec("C")];
    const right = [sec("B"), sec("A"), sec("C")];
    const result = matchSections(left, right);
    // LCS: B, C（長さ2）。A は左では B の前、右では B の後なので不一致
    const matched = result.filter(r => r.type === "matched");
    expect(matched.length).toBe(2);
  });
});
```

**Step 2: テスト実行して失敗を確認**

Run: `cd packages/editor-core && npx jest sectionParser --no-coverage`\
Expected: FAIL（`matchSections` が存在しない）

**Step 3: 実装**

```typescript
// sectionParser.ts に追加

export interface SectionMatch {
  type: "matched" | "left-only" | "right-only";
  left: MarkdownSection | null;
  right: MarkdownSection | null;
}

export function matchSections(
  leftSections: MarkdownSection[],
  rightSections: MarkdownSection[],
): SectionMatch[] {
  // 見出しテキストで LCS を計算
  const leftHeadings = leftSections.map(s => s.heading ?? "");
  const rightHeadings = rightSections.map(s => s.heading ?? "");
  const lcsIndices = computeLCS(leftHeadings, rightHeadings);

  const result: SectionMatch[] = [];
  let li = 0;
  let ri = 0;

  for (const [lIdx, rIdx] of lcsIndices) {
    // LCS ペアの前にある不一致を出力
    while (li < lIdx) { result.push({ type: "left-only", left: leftSections[li++], right: null }); }
    while (ri < rIdx) { result.push({ type: "right-only", left: null, right: rightSections[ri++] }); }
    result.push({ type: "matched", left: leftSections[lIdx], right: rightSections[rIdx] });
    li = lIdx + 1;
    ri = rIdx + 1;
  }
  // 残りを出力
  while (li < leftSections.length) { result.push({ type: "left-only", left: leftSections[li++], right: null }); }
  while (ri < rightSections.length) { result.push({ type: "right-only", left: null, right: rightSections[ri++] }); }

  return result;
}

/** 文字列配列の LCS インデックスペアを返す */
function computeLCS(a: string[], b: string[]): [number, number][] {
  const m = a.length;
  const n = b.length;
  const dp: number[][] = Array.from({ length: m + 1 }, () => Array(n + 1).fill(0));
  for (let i = 1; i <= m; i++) {
    for (let j = 1; j <= n; j++) {
      dp[i][j] = a[i - 1] === b[j - 1] ? dp[i - 1][j - 1] + 1 : Math.max(dp[i - 1][j], dp[i][j - 1]);
    }
  }
  // バックトラック
  const pairs: [number, number][] = [];
  let i = m, j = n;
  while (i > 0 && j > 0) {
    if (a[i - 1] === b[j - 1]) { pairs.push([i - 1, j - 1]); i--; j--; }
    else if (dp[i - 1][j] >= dp[i][j - 1]) { i--; }
    else { j--; }
  }
  return pairs.reverse();
}
```

**Step 4: テスト実行して通過を確認**

Run: `cd packages/editor-core && npx jest sectionParser --no-coverage`\
Expected: PASS

**Step 5: コミット**

```
git add packages/editor-core/src/utils/sectionParser.ts packages/editor-core/src/__tests__/sectionParser.test.ts
git commit -m "feat(editor-core): セクションLCSマッチング追加（TDD）"
```

---


### Task 3: `computeSemanticDiff` 実装（TDD）

**Files:**

- Modify: `packages/editor-core/src/utils/diffEngine.ts`
- Create: `packages/editor-core/src/__tests__/semanticDiff.test.ts`

**Step 1: テストを作成**

```typescript
// semanticDiff.test.ts
import { computeSemanticDiff } from "../utils/diffEngine";

describe("computeSemanticDiff", () => {
  test("同一テキストは全行 equal", () => {
    const text = "## A\nline1\n## B\nline2";
    const result = computeSemanticDiff(text, text);
    expect(result.blocks).toHaveLength(0);
    expect(result.leftLines.every(l => l.type === "equal")).toBe(true);
  });

  test("セクション内の変更が正しく検出される", () => {
    const left = "## A\nold line\n## B\nsame";
    const right = "## A\nnew line\n## B\nsame";
    const result = computeSemanticDiff(left, right);
    expect(result.blocks.length).toBeGreaterThan(0);
    expect(result.blocks[0].type).toBe("modified");
  });

  test("左にのみ存在するセクションがパディングされる", () => {
    const left = "## A\nbody a\n## B\nbody b";
    const right = "## A\nbody a";
    const result = computeSemanticDiff(left, right);
    // 右側に ## B セクション分のパディングが入る
    const paddingLines = result.rightLines.filter(l => l.type === "padding");
    expect(paddingLines.length).toBeGreaterThan(0);
  });

  test("右にのみ存在するセクションがパディングされる", () => {
    const left = "## A\nbody a";
    const right = "## A\nbody a\n## C\nbody c";
    const result = computeSemanticDiff(left, right);
    const paddingLines = result.leftLines.filter(l => l.type === "padding");
    expect(paddingLines.length).toBeGreaterThan(0);
  });

  test("サブ見出しが再帰的にマッチングされる", () => {
    const left = "## A\n### Sub1\nold\n### Sub2\nsame";
    const right = "## A\n### Sub1\nnew\n### Sub2\nsame";
    const result = computeSemanticDiff(left, right);
    expect(result.blocks.length).toBe(1);
    expect(result.blocks[0].type).toBe("modified");
  });

  test("見出しなしテキストは computeDiff にフォールバック", () => {
    const left = "line1\nline2";
    const right = "line1\nline3";
    const result = computeSemanticDiff(left, right);
    expect(result.blocks.length).toBeGreaterThan(0);
  });

  test("leftLines と rightLines の長さが一致する", () => {
    const left = "## A\na1\na2\n## B\nb1";
    const right = "## A\na1\n## C\nc1\n## B\nb1";
    const result = computeSemanticDiff(left, right);
    expect(result.leftLines.length).toBe(result.rightLines.length);
  });
});
```

**Step 2: テスト実行して失敗を確認**

Run: `cd packages/editor-core && npx jest semanticDiff --no-coverage`\
Expected: FAIL

**Step 3: `computeSemanticDiff` を実装**

`diffEngine.ts` の末尾に追加。\
`sectionParser.ts` の `parseMarkdownSections` と `matchSections` をインポートして使用。\
セクションごとに `computeDiff` を呼び出し、結果を結合する。\
`DiffResult` 型はそのまま維持。blockId は結合時にリナンバリング。

**Step 4: テスト実行して通過を確認**

Run: `cd packages/editor-core && npx jest semanticDiff --no-coverage`\
Expected: PASS

**Step 5: コミット**

```
git add packages/editor-core/src/utils/diffEngine.ts packages/editor-core/src/__tests__/semanticDiff.test.ts
git commit -m "feat(editor-core): computeSemanticDiff 実装（TDD）"
```

---


### Task 4: `useMergeDiff` に `semantic` オプション追加

**Files:**

- Modify: `packages/editor-core/src/hooks/useMergeDiff.ts`
- Modify: `packages/editor-core/src/utils/diffEngine.ts`（`DiffOptions` に `semantic` 追加）

**Step 1: `DiffOptions` に `semantic` フィールド追加**

```typescript
// diffEngine.ts
export interface DiffOptions {
  ignoreWhitespace?: boolean;
  ignoreCase?: boolean;
  ignoreBlankLines?: boolean;
  semantic?: boolean;  // 追加
}
```

**Step 2: `useMergeDiff` の diff 関数を切替**

```typescript
// useMergeDiff.ts line 29-32 を変更
const diffResult: DiffResult | null = useMemo(() => {
  if (editText === "" && compareText === "") return null;
  return diffOptions.semantic
    ? computeSemanticDiff(editText, compareText, diffOptions)
    : computeDiff(editText, compareText, diffOptions);
}, [editText, compareText, diffOptions]);
```

**Step 3: テスト実行**

Run: `cd packages/editor-core && npx jest --no-coverage`\
Expected: 全 PASS

**Step 4: コミット**

```
git add packages/editor-core/src/utils/diffEngine.ts packages/editor-core/src/hooks/useMergeDiff.ts
git commit -m "feat(editor-core): useMergeDiff に semantic オプション追加"
```

---


### Task 5: UI トグルボタン追加

**Files:**

- Modify: `packages/editor-core/src/components/InlineMergeView.tsx`
- Modify: `packages/editor-core/src/i18n/en.json`
- Modify: `packages/editor-core/src/i18n/ja.json`

**Step 1: i18n キー追加**

```json
// en.json
"semanticDiff": "Semantic",

// ja.json
"semanticDiff": "セマンティック",
```

**Step 2: `InlineMergeView` にトグル追加**

`useMergeDiff` から `diffOptions` と `setDiffOptions` を取得し、ツールバー領域にトグルボタンを配置。\
MUI `ToggleButton` で `semantic` の ON/OFF を切替。\
アイコンは `AccountTreeOutlined`（既存インポートまたは追加）。

**Step 3: ビルド・テスト実行**

Run: `npx tsc --noEmit && cd packages/editor-core && npx jest --no-coverage`\
Expected: 全 PASS

**Step 4: コミット**

```
git add packages/editor-core/src/components/InlineMergeView.tsx packages/editor-core/src/i18n/en.json packages/editor-core/src/i18n/ja.json
git commit -m "feat(editor-core): セマンティック比較トグルUI追加"
```

---


### Task 6: WYSIWYG モード対応

**Files:**

- Modify: `packages/editor-core/src/hooks/useDiffHighlight.ts`
- Modify: `packages/editor-core/src/extensions/diffHighlight.ts`

**Step 1: `useDiffHighlight` でセマンティック比較結果を使用**

WYSIWYG モードの `computeBlockDiff` に、セマンティック比較で得られたセクションマッチング情報を渡す。\
マッチしなかったセクションのトップレベルノードを一括ハイライト。\
マッチしたセクション内のノードは既存の `computeBlockDiff` で比較。

**Step 2: ビルド・テスト実行**

Run: `npx tsc --noEmit && cd packages/editor-core && npx jest --no-coverage`\
Expected: 全 PASS

**Step 3: web-app ビルド検証**

Run: `cd packages/web-app && npx next build`\
Expected: ビルド成功

**Step 4: コミット**

```
git add packages/editor-core/src/hooks/useDiffHighlight.ts packages/editor-core/src/extensions/diffHighlight.ts
git commit -m "feat(editor-core): WYSIWYGモードのセマンティック比較対応"
```

---


### Task 7: エクスポート・最終検証

**Files:**

- Modify: `packages/editor-core/src/index.ts`
- Modify: `packages/editor-core/src/exports/types.ts`

**Step 1: エクスポート追加**

```typescript
// index.ts
export { computeSemanticDiff } from './utils/diffEngine';

// exports/types.ts
export type { MarkdownSection, SectionMatch } from '../utils/sectionParser';
```

**Step 2: 全テスト実行**

Run: `cd packages/editor-core && npx jest --no-coverage`\
Expected: 全 PASS

**Step 3: web-app ビルド検証**

Run: `cd packages/web-app && npx next build`\
Expected: ビルド成功

**Step 4: vscode-extension 型チェック**

Run: `npx tsc --noEmit`\
Expected: エラーなし

**Step 5: コミット**

```
git add packages/editor-core/src/index.ts packages/editor-core/src/exports/types.ts
git commit -m "feat(editor-core): computeSemanticDiff をエクスポート追加"
```
