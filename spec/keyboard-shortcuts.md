# キーボードショートカット一覧

更新日: 2026-03-18

凡例:

- **Mod**: Windows/Linux では `Ctrl`、Mac では `Cmd`
- **Web**: Web アプリ（PC ブラウザ）
- **VS**: VS Code 拡張
- o: 利用可 / x: 利用不可 / △: 条件付き


## 1. ファイル操作

| ショートカット | 動作 | WYSIWYG | Source | Review | Readonly | Web | VS | 備考 |
| --- | --- | :---: | :---: | :---: | :---: | :---: | :---: | --- |
| Mod+S | 保存 | o | o | o | o | o | o | VS は VS Code ネイティブ保存 |
| Mod+Shift+S | 名前を付けて保存 | o | o | o | o | o | x | |
| Mod+O | ファイルを開く | o | o | o | o | o | x | |
| Mod+Alt+N | 新規作成（クリア） | o | o | x | x | o | x | |
| Mod+Alt+U | ファイルインポート | o | o | x | x | o | x | |
| Mod+Alt+E | ダウンロード/エクスポート | o | o | x | x | o | x | |


## 2. モード切替

| ショートカット | 動作 | WYSIWYG | Source | Review | Readonly | Web | VS | 備考 |
| --- | --- | :---: | :---: | :---: | :---: | :---: | :---: | --- |
| Mod+Alt+S | モード順送り切替 | o | o | o | o | o | o | Readonly→Review→WYSIWYG→Source→… |


## 3. 編集操作

| ショートカット | 動作 | WYSIWYG | Source | Review | Readonly | Web | VS | 備考 |
| --- | --- | :---: | :---: | :---: | :---: | :---: | :---: | --- |
| Mod+Z | 元に戻す | o | o | x | x | o | o | VS ソースモードは vscode-set-content 経由 |
| Mod+Y / Mod+Shift+Z | やり直し | o | o | x | x | o | o | |
| Mod+Shift+K | 行（ブロック）削除 | o | x | x | x | o | o | |
| Mod+Enter | 下に空行を挿入 | o | x | x | x | o | o | 現在のブロックの下に空の段落を挿入 |
| Mod+Shift+Enter | 上に空行を挿入 | o | x | x | x | o | o | 現在のブロックの上に空の段落を挿入 |
| Mod+L | 行（ブロック）選択 | o | x | x | x | o | o | トップレベルブロック全体を選択 |
| Mod+D | 単語選択 | o | x | x | x | o | o | カーソル位置の単語を選択（日本語対応） |
| Shift+Enter | ハードブレイク | o | x | x | x | o | o | コードブロック内では無効 |


## 4. ブロック操作

| ショートカット | 動作 | WYSIWYG | Source | Review | Readonly | Web | VS | 備考 |
| --- | --- | :---: | :---: | :---: | :---: | :---: | :---: | --- |
| Alt+Up | ブロックを上に移動 | o | x | x | x | o | o | |
| Alt+Down | ブロックを下に移動 | o | x | x | x | o | o | |
| Shift+Alt+Up | ブロックを上に複製 | o | x | x | x | o | o | |
| Shift+Alt+Down | ブロックを下に複製 | o | x | x | x | o | o | |
| Mod+Alt+F | 全ブロック展開/折りたたみ | o | x | x | x | o | o | |


## 5. 見出し操作

| ショートカット | 動作 | WYSIWYG | Source | Review | Readonly | Web | VS | 備考 |
| --- | --- | :---: | :---: | :---: | :---: | :---: | :---: | --- |
| Tab | 見出しレベルを下げる | △ | x | x | x | o | o | 見出し内でのみ有効（H1→H2→…→H5） |
| Shift+Tab | 見出しレベルを上げる | △ | x | x | x | o | o | 見出し内でのみ有効（H5→H4→…→H1） |


## 6. 検索・置換

| ショートカット | 動作 | WYSIWYG | Source | Review | Readonly | Web | VS | 備考 |
| --- | --- | :---: | :---: | :---: | :---: | :---: | :---: | --- |
| Mod+F | 検索パネルを開く | o | o | o | o | o | o | ソースモードはソース検索バー |
| Mod+H | 検索・置換パネルを開く | o | x | o | o | o | o | |
| Enter | 次のマッチへ | △ | △ | △ | △ | o | o | 検索バーにフォーカス時 |
| Shift+Enter | 前のマッチへ | △ | △ | △ | △ | o | o | 検索バーにフォーカス時 |
| Escape | 検索パネルを閉じる | △ | △ | △ | △ | o | o | 検索バーにフォーカス時 |


## 7. 挿入操作

| ショートカット | 動作 | WYSIWYG | Source | Review | Readonly | Web | VS | 備考 |
| --- | --- | :---: | :---: | :---: | :---: | :---: | :---: | --- |
| Mod+Alt+I | 画像挿入 | o | x | x | x | o | o | |
| Mod+Alt+R | 水平線挿入 | o | x | x | x | o | o | |
| Mod+Alt+T | テーブル挿入（3x3） | o | x | x | x | o | o | |
| Mod+Alt+D | ダイアグラムメニュー | o | x | x | x | o | o | |
| Mod+Alt+P | テンプレートメニュー | o | x | x | x | o | o | |
| Mod+Alt+M | 比較モード切替 | o | o | o | x | o | o | |


## 8. 貼り付け

| ショートカット | 動作 | WYSIWYG | Source | Review | Readonly | Web | VS | 備考 |
| --- | --- | :---: | :---: | :---: | :---: | :---: | :---: | --- |
| Mod+V | 貼り付け | o | o | x | x | o | o | |
| Mod+Shift+V | Markdown で貼り付け | o | x | x | x | o | o | 罫線テーブル自動変換対応 |


## 9. スラッシュコマンド

| ショートカット | 動作 | WYSIWYG | Source | Review | Readonly | Web | VS | 備考 |
| --- | --- | :---: | :---: | :---: | :---: | :---: | :---: | --- |
| / | スラッシュコマンドメニュー表示 | o | x | x | x | o | o | ブロック先頭で入力 |
| Up / Down | コマンド選択 | △ | x | x | x | o | o | メニュー表示中 |
| Enter | コマンド実行 | △ | x | x | x | o | o | メニュー表示中 |
| Escape | メニューを閉じる | △ | x | x | x | o | o | メニュー表示中 |


## 10. コードブロック内

| ショートカット | 動作 | WYSIWYG | Source | Review | Readonly | Web | VS | 備考 |
| --- | --- | :---: | :---: | :---: | :---: | :---: | :---: | --- |
| Escape | コードブロックから脱出 | o | x | x | x | o | o | カーソルを次のブロックに移動 |
| Up / Down | 隣接コードブロック間移動 | o | x | x | x | o | o | コードブロックの端で有効 |
| Tab | タブ文字挿入 | x | o | x | x | o | o | ソースモードのみ |


## 11. 画像リサイズ

| ショートカット | 動作 | WYSIWYG | Source | Review | Readonly | Web | VS | 備考 |
| --- | --- | :---: | :---: | :---: | :---: | :---: | :---: | --- |
| Right | 幅を +10px | △ | x | x | x | o | o | リサイズハンドルにフォーカス時 |
| Left | 幅を -10px | △ | x | x | x | o | o | リサイズハンドルにフォーカス時 |
| Shift+Right | 幅を +50px | △ | x | x | x | o | o | リサイズハンドルにフォーカス時 |
| Shift+Left | 幅を -50px | △ | x | x | x | o | o | リサイズハンドルにフォーカス時 |


## 12. ダイアログ

| ショートカット | 動作 | WYSIWYG | Source | Review | Readonly | Web | VS | 備考 |
| --- | --- | :---: | :---: | :---: | :---: | :---: | :---: | --- |
| Mod+Enter | コメント送信 | △ | x | x | x | o | o | コメントダイアログ内 |
| Enter | リンク/画像挿入確定 | △ | x | x | x | o | o | URL 入力欄内 |
| Escape | ダイアログを閉じる | o | x | o | o | o | o | |


## 13. コメントパネル

| ショートカット | 動作 | WYSIWYG | Source | Review | Readonly | Web | VS | 備考 |
| --- | --- | :---: | :---: | :---: | :---: | :---: | :---: | --- |
| Mod+Enter | コメント編集を確定 | △ | x | △ | x | o | o | コメント編集中 |
| Escape | コメント編集をキャンセル | △ | x | △ | x | o | o | コメント編集中 |


## 14. アウトラインパネル

| ショートカット | 動作 | WYSIWYG | Source | Review | Readonly | Web | VS | 備考 |
| --- | --- | :---: | :---: | :---: | :---: | :---: | :---: | --- |
| Alt+Up | 見出しを上に移動 | △ | x | x | x | o | o | 見出し項目にフォーカス時 |
| Alt+Down | 見出しを下に移動 | △ | x | x | x | o | o | 見出し項目にフォーカス時 |
| Right | パネル幅を +20px | △ | △ | △ | △ | o | x | リサイズハンドルにフォーカス時 |
| Left | パネル幅を -20px | △ | △ | △ | △ | o | x | リサイズハンドルにフォーカス時 |


## 15. 無効化されたショートカット

以下のショートカットは意図的に無効化されています。\
バブルメニュー・ツールバー・スラッシュコマンドから操作してください。

| ショートカット | 本来の動作 |
| --- | --- |
| Mod+B | 太字 |
| Mod+I | 斜体 |
| Mod+U | 下線 |
| Mod+E | インラインコード |
| Mod+Shift+X | 取消線 |
| Mod+Shift+H | ハイライト |
| Mod+Shift+7 | 番号付きリスト |
| Mod+Shift+8 | 箇条書きリスト |
| Mod+Shift+9 | タスクリスト |
| Mod+K | リンク挿入 |


## 16. 右クリックメニュー

右クリックで表示されるコンテキストメニューの操作。

| メニュー | ショートカット | 編集モード | Review/Readonly | Web | VS |
| --- | --- | :---: | :---: | :---: | :---: |
| 切り取り | Mod+X | o（選択あり） | x | o | o |
| コピー | Mod+C | o（選択あり） | o（選択あり） | o | o |
| 貼り付け | Mod+V | o | x | o | o |
| Markdown で貼り付け | Mod+Shift+V | o | x | o | o |
| コードブロックで貼り付け | — | o | x | o | o |
