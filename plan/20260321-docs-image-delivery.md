# S3フォルダ構成での画像配信 実装計画

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** S3の docs/ 配下にフォルダ構成で配置したmdファイル内の相対パス画像を、docs/viewページで正しく表示する。

**Architecture:** `/api/docs/content` でmd返却前に相対パスを絶対URLに変換する。CloudFront設定時はCDN直接URL、未設定時は新規 `/api/docs/image` API経由のURLに変換する。画像パス変換ロジックは純粋関数として切り出しテスト容易にする。

**Tech Stack:** Next.js App Router, AWS S3 SDK, TypeScript, Jest

---

## S3上の想定構成

```
docs/
  _layout.json
  claude-code-markdown-format-spec.md     ← 既存フラット（画像なし、変換不要）
  landing-guide/                           ← 新規フォルダ構成
    index.md
    images/
      screenshot1.png
      demo.gif
```

---

### Task 1: 画像パス変換の純粋関数を作成

**Files:**

- Create: `packages/web-app/src/lib/__tests__/docsImageUrl.test.ts`
- Create: `packages/web-app/src/lib/docsImageUrl.ts`

**Step 1: テストファイルを作成**

```typescript
// packages/web-app/src/lib/__tests__/docsImageUrl.test.ts
import { resolveImageUrl, transformMarkdownImageUrls } from '../docsImageUrl';

describe('resolveImageUrl', () => {
  it('相対パスをCDN URLに変換する', () => {
    expect(
      resolveImageUrl('./images/screenshot.png', 'docs/guide/', 'https://cdn.example.com')
    ).toBe('https://cdn.example.com/docs/guide/images/screenshot.png');
  });

  it('./ なしの相対パスも変換する', () => {
    expect(
      resolveImageUrl('images/screenshot.png', 'docs/guide/', 'https://cdn.example.com')
    ).toBe('https://cdn.example.com/docs/guide/images/screenshot.png');
  });

  it('CloudFront未設定時はAPI URLにフォールバック', () => {
    expect(
      resolveImageUrl('./images/screenshot.png', 'docs/guide/', '')
    ).toBe('/api/docs/image?key=docs%2Fguide%2Fimages%2Fscreenshot.png');
  });

  it('絶対URLはそのまま返す', () => {
    expect(
      resolveImageUrl('https://example.com/img.png', 'docs/guide/', 'https://cdn.example.com')
    ).toBe('https://example.com/img.png');
  });

  it('data URLはそのまま返す', () => {
    expect(
      resolveImageUrl('data:image/png;base64,abc', 'docs/guide/', 'https://cdn.example.com')
    ).toBe('data:image/png;base64,abc');
  });

  it('フラットなmdファイル（baseDir = "docs/"）でも動作する', () => {
    expect(
      resolveImageUrl('./images/foo.png', 'docs/', 'https://cdn.example.com')
    ).toBe('https://cdn.example.com/docs/images/foo.png');
  });
});

describe('transformMarkdownImageUrls', () => {
  const baseDir = 'docs/guide/';
  const cdnUrl = 'https://cdn.example.com';

  it('Markdown画像構文の相対パスを変換する', () => {
    const input = '# Title\n\n![説明](./images/screenshot.png)\n\ntext';
    const result = transformMarkdownImageUrls(input, baseDir, cdnUrl);
    expect(result).toBe(
      '# Title\n\n![説明](https://cdn.example.com/docs/guide/images/screenshot.png)\n\ntext'
    );
  });

  it('HTMLのimg srcも変換する', () => {
    const input = '<img src="./images/demo.gif" alt="demo">';
    const result = transformMarkdownImageUrls(input, baseDir, cdnUrl);
    expect(result).toBe(
      '<img src="https://cdn.example.com/docs/guide/images/demo.gif" alt="demo">'
    );
  });

  it('絶対URLは変換しない', () => {
    const input = '![img](https://example.com/img.png)';
    const result = transformMarkdownImageUrls(input, baseDir, cdnUrl);
    expect(result).toBe('![img](https://example.com/img.png)');
  });

  it('複数の画像を一括変換する', () => {
    const input = '![a](./images/a.png)\n![b](./images/b.jpg)';
    const result = transformMarkdownImageUrls(input, baseDir, cdnUrl);
    expect(result).toContain('https://cdn.example.com/docs/guide/images/a.png');
    expect(result).toContain('https://cdn.example.com/docs/guide/images/b.jpg');
  });

  it('画像がないmdはそのまま返す', () => {
    const input = '# Title\n\nsome text';
    const result = transformMarkdownImageUrls(input, baseDir, cdnUrl);
    expect(result).toBe(input);
  });
});
```

**Step 2: テストが失敗することを確認**

Run: `cd packages/web-app && npx jest src/lib/__tests__/docsImageUrl.test.ts --no-coverage`
Expected: FAIL（モジュールが存在しない）

**Step 3: 実装を作成**

```typescript
// packages/web-app/src/lib/docsImageUrl.ts

/**
 * md ファイルの key から画像パス解決用の baseDir を算出する。
 * 例: "docs/guide/index.md" → "docs/guide/"
 *     "docs/flat-file.md"   → "docs/"
 */
export function getBaseDir(key: string): string {
  const lastSlash = key.lastIndexOf('/');
  return lastSlash >= 0 ? key.slice(0, lastSlash + 1) : '';
}

/**
 * 単一の画像URLを解決する。
 * - 絶対URL / data URL → そのまま返す
 * - 相対パス + CloudFront設定あり → CDN URL
 * - 相対パス + CloudFront未設定 → /api/docs/image?key=... URL
 */
export function resolveImageUrl(
  src: string,
  baseDir: string,
  cloudfrontUrl: string,
): string {
  // 絶対URL・data URL・アンカーはスキップ
  if (/^(https?:\/\/|data:|#)/.test(src)) return src;

  // "./" プレフィックスを除去
  const relativePath = src.replace(/^\.\//, '');
  const imageKey = baseDir + relativePath;

  if (cloudfrontUrl) {
    // 末尾スラッシュを正規化
    const base = cloudfrontUrl.replace(/\/+$/, '');
    return `${base}/${imageKey}`;
  }

  return `/api/docs/image?key=${encodeURIComponent(imageKey)}`;
}

/**
 * Markdown テキスト内の画像相対パスを絶対URLに一括変換する。
 * 対象: ![alt](src) と <img src="src">
 */
export function transformMarkdownImageUrls(
  markdown: string,
  baseDir: string,
  cloudfrontUrl: string,
): string {
  // Markdown: ![alt](src) — src 部分のみ置換
  let result = markdown.replace(
    /(!\[[^\]]*\]\()([^)]+)(\))/g,
    (_match, prefix, src, suffix) =>
      prefix + resolveImageUrl(src, baseDir, cloudfrontUrl) + suffix,
  );

  // HTML: <img src="src"> — src 属性のみ置換（ダブルクオート・シングルクオート両対応）
  result = result.replace(
    /(<img\s[^>]*src=)(["'])([^"']+)\2/g,
    (_match, prefix, quote, src) =>
      prefix + quote + resolveImageUrl(src, baseDir, cloudfrontUrl) + quote,
  );

  return result;
}
```

**Step 4: テストが通ることを確認**

Run: `cd packages/web-app && npx jest src/lib/__tests__/docsImageUrl.test.ts --no-coverage`
Expected: PASS

**Step 5: コミット**

```bash
git add packages/web-app/src/lib/docsImageUrl.ts packages/web-app/src/lib/__tests__/docsImageUrl.test.ts
git commit -m "feat(web-app): docs画像パス変換の純粋関数を追加"
```

---

### Task 2: /api/docs/content に画像パス変換を組み込む

**Files:**

- Modify: `packages/web-app/src/lib/s3Client.ts` — `CLOUDFRONT_URL` を export する
- Modify: `packages/web-app/src/app/api/docs/content/route.ts` — md返却前にパス変換を適用

**Step 1: s3Client.ts で CLOUDFRONT_URL を export**

`packages/web-app/src/lib/s3Client.ts` L19 を変更:

```diff
- const CLOUDFRONT_URL = process.env.CLOUDFRONT_DOCS_URL ?? '';
+ export const CLOUDFRONT_URL = process.env.CLOUDFRONT_DOCS_URL ?? '';
```

**Step 2: content/route.ts に変換処理を追加**

`packages/web-app/src/app/api/docs/content/route.ts` を変更:

```typescript
import { GetObjectCommand } from '@aws-sdk/client-s3';
import { NextRequest, NextResponse } from 'next/server';

import { getBaseDir, transformMarkdownImageUrls } from '../../../../lib/docsImageUrl';
import { CLOUDFRONT_URL, DOCS_BUCKET, DOCS_PREFIX, fetchFromCdn, s3Client } from '../../../../lib/s3Client';

export const dynamic = 'force-dynamic';

export async function GET(request: NextRequest) {
  const key = request.nextUrl.searchParams.get('key');

  if (!key) {
    return NextResponse.json({ error: 'key parameter is required' }, { status: 400 });
  }

  if (!key.startsWith(DOCS_PREFIX) || !key.endsWith('.md')) {
    return NextResponse.json({ error: 'Invalid key' }, { status: 400 });
  }

  // パストラバーサル防止
  if (key.includes('..')) {
    return NextResponse.json({ error: 'Invalid key' }, { status: 400 });
  }

  const cacheHeaders = {
    'Content-Type': 'text/markdown; charset=utf-8',
    'Cache-Control': 'private, max-age=3600, stale-while-revalidate=86400',
    'CDN-Cache-Control': 'no-store',
    'Netlify-CDN-Cache-Control': 'no-store',
  };

  try {
    let body: string | null = null;

    // CloudFront 経由で取得を試みる
    body = await fetchFromCdn(key);

    // S3 SDK フォールバック
    if (body === null) {
      if (!DOCS_BUCKET) {
        return NextResponse.json(
          { error: 'S3_DOCS_BUCKET is not configured' },
          { status: 500 },
        );
      }

      const command = new GetObjectCommand({
        Bucket: DOCS_BUCKET,
        Key: key,
      });
      const response = await s3Client.send(command);
      body = (await response.Body?.transformToString('utf-8')) ?? null;
    }

    if (!body) {
      return NextResponse.json({ error: 'Empty document' }, { status: 404 });
    }

    // 画像の相対パスを絶対URLに変換
    const baseDir = getBaseDir(key);
    body = transformMarkdownImageUrls(body, baseDir, CLOUDFRONT_URL);

    return new NextResponse(body, { status: 200, headers: cacheHeaders });
  } catch (e: unknown) {
    if (e instanceof Error && e.name === 'NoSuchKey') {
      return NextResponse.json({ error: 'Document not found' }, { status: 404 });
    }
    console.error('Failed to get S3 object:', e);
    return NextResponse.json(
      { error: 'Failed to load document' },
      { status: 500 },
    );
  }
}
```

**Step 3: ビルド確認**

Run: `cd packages/web-app && npx next build`
Expected: ビルド成功

**Step 4: コミット**

```bash
git add packages/web-app/src/lib/s3Client.ts packages/web-app/src/app/api/docs/content/route.ts
git commit -m "feat(web-app): docs content APIに画像パス変換を組み込み"
```

---

### Task 3: /api/docs/image エンドポイントを新規作成

**Files:**

- Create: `packages/web-app/src/app/api/docs/image/route.ts`

**Step 1: 画像配信APIを作成**

```typescript
// packages/web-app/src/app/api/docs/image/route.ts
import { GetObjectCommand } from '@aws-sdk/client-s3';
import { NextRequest, NextResponse } from 'next/server';

import { DOCS_BUCKET, DOCS_PREFIX, s3Client } from '../../../../lib/s3Client';

export const dynamic = 'force-dynamic';

const ALLOWED_EXTENSIONS: Record<string, string> = {
  '.png': 'image/png',
  '.jpg': 'image/jpeg',
  '.jpeg': 'image/jpeg',
  '.gif': 'image/gif',
  '.webp': 'image/webp',
  '.svg': 'image/svg+xml',
};

export async function GET(request: NextRequest) {
  const key = request.nextUrl.searchParams.get('key');

  if (!key) {
    return NextResponse.json({ error: 'key parameter is required' }, { status: 400 });
  }

  if (!key.startsWith(DOCS_PREFIX)) {
    return NextResponse.json({ error: 'Invalid key' }, { status: 400 });
  }

  // パストラバーサル防止
  if (key.includes('..')) {
    return NextResponse.json({ error: 'Invalid key' }, { status: 400 });
  }

  // 拡張子チェック
  const ext = key.slice(key.lastIndexOf('.')).toLowerCase();
  const contentType = ALLOWED_EXTENSIONS[ext];
  if (!contentType) {
    return NextResponse.json({ error: 'Unsupported image format' }, { status: 400 });
  }

  if (!DOCS_BUCKET) {
    return NextResponse.json(
      { error: 'S3_DOCS_BUCKET is not configured' },
      { status: 500 },
    );
  }

  try {
    const command = new GetObjectCommand({
      Bucket: DOCS_BUCKET,
      Key: key,
    });
    const response = await s3Client.send(command);
    const bodyBytes = await response.Body?.transformToByteArray();

    if (!bodyBytes) {
      return NextResponse.json({ error: 'Image not found' }, { status: 404 });
    }

    return new NextResponse(bodyBytes, {
      status: 200,
      headers: {
        'Content-Type': contentType,
        'Cache-Control': 'public, max-age=86400, stale-while-revalidate=604800',
        'CDN-Cache-Control': 'public, max-age=86400',
      },
    });
  } catch (e: unknown) {
    if (e instanceof Error && e.name === 'NoSuchKey') {
      return NextResponse.json({ error: 'Image not found' }, { status: 404 });
    }
    console.error('Failed to get S3 image:', e);
    return NextResponse.json(
      { error: 'Failed to load image' },
      { status: 500 },
    );
  }
}
```

**Step 2: ビルド確認**

Run: `cd packages/web-app && npx next build`
Expected: ビルド成功

**Step 3: コミット**

```bash
git add packages/web-app/src/app/api/docs/image/route.ts
git commit -m "feat(web-app): docs画像配信APIを追加（S3フォールバック用）"
```

---

### Task 4: 動作確認

**Step 1: S3にテスト用フォルダ構成をアップロード**

手動作業: S3バケットの `docs/` 配下にテスト用フォルダを作成し、mdと画像をアップロードする。

```
docs/
  test-guide/
    index.md        ← ![テスト画像](./images/test.png) を含む
    images/
      test.png
```

**Step 2: ブラウザで確認**

URL: `https://www.anytime-trial.com/docs/view?key=docs%2Ftest-guide%2Findex.md`

確認項目:
- [ ] mdの本文が正しく表示される
- [ ] 画像が正しく表示される（CloudFront URL or API URL）
- [ ] ブラウザのDevToolsで画像のsrcが期待通りのURLになっている
- [ ] 既存のフラットなmdファイル（`docs/claude-code-markdown-format-spec.md`）が引き続き正常に表示される

**Step 3: ローカル環境での確認（CloudFront未設定）**

CLOUDFRONT_DOCS_URL を空にした状態で動作確認し、`/api/docs/image` 経由で画像が取得できることを確認する。
