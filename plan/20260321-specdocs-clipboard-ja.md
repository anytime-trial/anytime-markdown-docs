# SpecDocs TreeView 切り取り/コピー/貼り付け 実装計画

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** VS Code拡張のSpecDocs TreeViewにCtrl+X/Ctrl+C/Ctrl+Vとコンテキストメニューで切り取り・コピー・貼り付け機能を追加する。

**Architecture:** SpecDocsProviderに内部クリップボード状態（paths + isCut）を持ち、cut/copy/pasteメソッドを追加。package.jsonにコマンド定義・コンテキストメニュー・キーバインドを追加し、extension.tsでコマンド登録する。

**Tech Stack:** VS Code Extension API, TypeScript, Node.js fs

---

### Task 1: package.json にコマンド定義・メニュー・キーバインドを追加

**Files:**

- Modify: `packages/vscode-extension/package.json`
- Modify: `packages/vscode-extension/package.nls.json`
- Modify: `packages/vscode-extension/package.nls.ja.json`

**Step 1: package.nls.json にラベルを追加**

L41 の `"command.pasteAsMarkdown"` の前に追加:

```json
  "command.specDocsCut": "Cut",
  "command.specDocsCopy": "Copy",
  "command.specDocsPaste": "Paste",
```

**Step 2: package.nls.ja.json にラベルを追加**

L41 の `"command.pasteAsMarkdown"` の前に追加:

```json
  "command.specDocsCut": "切り取り",
  "command.specDocsCopy": "コピー",
  "command.specDocsPaste": "貼り付け",
```

**Step 3: package.json の `contributes.commands` にコマンド定義を追加**

`specDocsImportFiles` コマンド定義の後に追加:

```json
,
{
  "command": "anytime-markdown.specDocsCut",
  "title": "%command.specDocsCut%",
  "icon": "$(cut)"
},
{
  "command": "anytime-markdown.specDocsCopy",
  "title": "%command.specDocsCopy%",
  "icon": "$(copy)"
},
{
  "command": "anytime-markdown.specDocsPaste",
  "title": "%command.specDocsPaste%",
  "icon": "$(paste)"
}
```

**Step 4: package.json の `view/item/context` にメニュー項目を追加**

既存の `2_edit` グループ（Rename, Delete, Copy Path の間）に切り取り・コピー・貼り付けを追加。
`specDocsRename` の前（`2_edit` グループの先頭）に挿入:

```json
{
  "command": "anytime-markdown.specDocsCut",
  "when": "view == anytimeMarkdown.specDocs && viewItem != specDocsRoot",
  "group": "2_edit@0a"
},
{
  "command": "anytime-markdown.specDocsCopy",
  "when": "view == anytimeMarkdown.specDocs && viewItem != specDocsRoot",
  "group": "2_edit@0b"
},
{
  "command": "anytime-markdown.specDocsPaste",
  "when": "view == anytimeMarkdown.specDocs && viewItem != specDocsRoot",
  "group": "2_edit@0c"
},
```

さらに specDocsRoot にも貼り付けを追加（ルートフォルダへの貼り付け用）:

```json
{
  "command": "anytime-markdown.specDocsPaste",
  "when": "view == anytimeMarkdown.specDocs && viewItem == specDocsRoot",
  "group": "1_new@4"
},
```

**Step 5: package.json の `keybindings` にショートカットを追加**

既存の `keybindings` 配列に追加:

```json
{
  "command": "anytime-markdown.specDocsCut",
  "key": "ctrl+x",
  "mac": "cmd+x",
  "when": "focusedView == anytimeMarkdown.specDocs"
},
{
  "command": "anytime-markdown.specDocsCopy",
  "key": "ctrl+c",
  "mac": "cmd+c",
  "when": "focusedView == anytimeMarkdown.specDocs"
},
{
  "command": "anytime-markdown.specDocsPaste",
  "key": "ctrl+v",
  "mac": "cmd+v",
  "when": "focusedView == anytimeMarkdown.specDocs"
}
```

**Step 6: コミット**

```bash
git add packages/vscode-extension/package.json packages/vscode-extension/package.nls.json packages/vscode-extension/package.nls.ja.json
git commit -m "feat(vscode-extension): TreeViewの切り取り/コピー/貼り付けコマンド定義を追加"
```

---

### Task 2: SpecDocsProvider に cut/copy/paste メソッドを追加

**Files:**

- Modify: `packages/vscode-extension/src/providers/SpecDocsProvider.ts`

**Step 1: クリップボード状態のプロパティを追加**

`SpecDocsProvider` クラスの `private _mdOnly: boolean;` の後に追加:

```typescript
private clipboard: { paths: string[]; isCut: boolean } | null = null;
```

**Step 2: cut メソッドを追加**

`toggleMdOnly()` メソッドの後に追加:

```typescript
cut(item: SpecDocsItem): void {
	this.clipboard = { paths: [item.resourceUri.fsPath], isCut: true };
}

copy(item: SpecDocsItem): void {
	this.clipboard = { paths: [item.resourceUri.fsPath], isCut: false };
}

async paste(item?: SpecDocsRootItem | SpecDocsItem): Promise<void> {
	if (!this.clipboard || this.clipboard.paths.length === 0) return;

	const destDir = this.resolveDestDir(item);
	if (!destDir) return;

	for (const srcPath of this.clipboard.paths) {
		const name = path.basename(srcPath);
		const dest = path.join(destDir, name);
		if (srcPath === dest) continue;

		if (fs.existsSync(dest)) {
			const answer = await vscode.window.showWarningMessage(
				`"${name}" already exists. Overwrite?`,
				{ modal: true },
				'Overwrite',
			);
			if (answer !== 'Overwrite') continue;
		}

		try {
			if (this.clipboard.isCut) {
				fs.renameSync(srcPath, dest);
			} else {
				if (fs.statSync(srcPath).isDirectory()) {
					fs.cpSync(srcPath, dest, { recursive: true });
				} else {
					fs.copyFileSync(srcPath, dest);
				}
			}
		} catch (e: unknown) {
			showError(this.clipboard.isCut ? 'Move failed' : 'Copy failed', e);
		}
	}

	if (this.clipboard.isCut) {
		this.clipboard = null;
	}
	this.refresh();
}
```

**Step 3: ビルド確認**

Run: `cd packages/vscode-extension && npx webpack --mode development 2>&1 | tail -5`
Expected: コンパイル成功

**Step 4: コミット**

```bash
git add packages/vscode-extension/src/providers/SpecDocsProvider.ts
git commit -m "feat(vscode-extension): SpecDocsProviderにcut/copy/pasteメソッドを追加"
```

---

### Task 3: extension.ts にコマンド登録を追加

**Files:**

- Modify: `packages/vscode-extension/src/extension.ts`

**Step 1: コマンド登録を追加**

`specDocsImportFiles` コマンド登録（L323-325）の後に追加:

```typescript
const specDocsCut = vscode.commands.registerCommand(
	'anytime-markdown.specDocsCut', (item: SpecDocsItem) => specDocsProvider.cut(item)
);
const specDocsCopy = vscode.commands.registerCommand(
	'anytime-markdown.specDocsCopy', (item: SpecDocsItem) => specDocsProvider.copy(item)
);
const specDocsPaste = vscode.commands.registerCommand(
	'anytime-markdown.specDocsPaste', (item?: SpecDocsRootItem | SpecDocsItem) => specDocsProvider.paste(item)
);
```

**Step 2: subscriptions に追加**

L457 の `specDocsImportFiles` の後に `, specDocsCut, specDocsCopy, specDocsPaste` を追加:

```typescript
specDocsCreateFile, specDocsCreateFolder, specDocsDelete, specDocsRename, specDocsRemoveRoot, specDocsCopyPath, specDocsImportFiles, specDocsCut, specDocsCopy, specDocsPaste, pasteAsMarkdown,
```

**Step 3: ビルド確認**

Run: `cd packages/vscode-extension && npx webpack --mode development 2>&1 | tail -5`
Expected: コンパイル成功

**Step 4: コミット**

```bash
git add packages/vscode-extension/src/extension.ts
git commit -m "feat(vscode-extension): cut/copy/pasteコマンドを登録"
```
