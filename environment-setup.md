# Anytime Markdown 環境構築ガイド

Tiptap ベースのリッチマークダウンエディタ。\
Web アプリ、VS Code 拡張機能、Android アプリの 3 プラットフォームで動作する。


## プロジェクト構成

npm workspaces によるモノレポ構成。

| パッケージ | 説明 | 依存先 |
| --- | --- | --- |
| `packages/editor-core` | 共有エディタライブラリ（Tiptap / React） | --- |
| `packages/web-app` | Next.js 15 Web アプリ | editor-core |
| `packages/vscode-extension` | VS Code 拡張機能 | editor-core |
| `packages/mobile-app` | Capacitor 7 Android アプリ | web-app（静的ビルド） |


## 前提条件

| ツール | 用途 |
| --- | --- |
| WSL2（Windows の場合） | Linux 環境 |
| Docker Desktop（WSL2 バックエンド） | コンテナ実行 |
| VS Code + Dev Containers 拡張機能 | 開発環境 |
| Android Studio + JDK 21 | Android アプリビルド時のみ |

> コンテナ内の Node.js は Dockerfile で Node 24 が使用される。\
> CI（GitHub Actions）は Node 22 で動作する。


## ホスト側の準備

Dev Container はホスト側の以下のディレクトリをマウントする。\
起動前に存在を確認すること。

| パス | 内容 |
| --- | --- |
| `~/.ssh` | Git の SSH 鍵 |
| `~/.claude` | Claude Code 設定 |
| `~/anytime-markdown-docs/` | ドキュメントリポジトリ |


## Dev Container でのセットアップ（推奨）

1. WSL2 上でリポジトリをクローンする

```bash
git clone git@github.com:anytime-trial/anytime-markdown.git
cd anytime-markdown
```

2. VS Code でリポジトリを開く
3. コマンドパレット → 「Dev Containers: Reopen in Container」を実行

> 初回はコンテナのビルドと `npm install` が自動実行される。\
> ポート `3000` は自動フォワードされる。

4. Web アプリの開発サーバーを起動する

```bash
cd packages/web-app
npm run dev
```

ブラウザで `http://localhost:3000` にアクセスする。


### GitHub Personal Access Token の設定（任意）

GitHub MCP サーバーや `gh` CLI で使用する。\
未設定でも開発は可能だが、PR 作成等の GitHub 操作が制限される。

1. https://github.com/settings/tokens で「Generate new token (classic)」→ スコープ `repo` にチェック
2. WSL のシェル設定ファイルに追加する

```bash
echo 'export GITHUB_PERSONAL_ACCESS_TOKEN=ghp_xxxxxxxxxxxxxxxx' >> ~/.bashrc
source ~/.bashrc
```

Dev Container 起動時にこの変数が設定されていれば、GitHub MCP サーバーが自動登録される。


## Docker を手動で使う場合

```bash
# コンテナをビルド・起動
docker compose up -d

# コンテナ内に入る
docker compose exec anytime-markdown bash

# 依存パッケージをインストール
npm install

# 開発サーバーを起動
cd packages/web-app
npm run dev
```


## 環境変数

`packages/web-app/.env.local.example` を `.env.local` にコピーして使用する。

```bash
cp packages/web-app/.env.local.example packages/web-app/.env.local
```

ローカル開発では環境変数なしで起動可能。\
以下の機能を使う場合のみ設定が必要。

| 機能 | 必要な変数 |
| --- | --- |
| Google Analytics | `NEXT_PUBLIC_GA_ID` |
| S3 ストレージ | `ANYTIME_AWS_*`, `S3_DOCS_BUCKET` |
| GitHub OAuth 連携 | `GITHUB_CLIENT_ID`, `GITHUB_CLIENT_SECRET`, `NEXTAUTH_*` |


## 主要な npm スクリプト


### ルート

| コマンド | 内容 |
| --- | --- |
| `npm run lint` | ESLint 実行（editor-core + web-app） |
| `npm run lint:fix` | ESLint 自動修正 |
| `npm run version:sync` | 全パッケージのバージョンを同期 |


### web-app (`packages/web-app`)

| コマンド | 内容 |
| --- | --- |
| `npm run dev` | 開発サーバー起動（port 3000） |
| `npm run build` | プロダクションビルド |
| `npm run e2e` | Playwright E2E テスト |
| `npm run e2e:ui` | Playwright UI モードで E2E テスト |
| `npm run analyze` | バンドルサイズ分析 |


### editor-core (`packages/editor-core`)

| コマンド | 内容 |
| --- | --- |
| `npm test` | Jest ユニットテスト |


### vscode-extension (`packages/vscode-extension`)

| コマンド | 内容 |
| --- | --- |
| `npm run compile` | webpack ビルド |
| `npm run watch` | webpack ウォッチモード |
| `npm run package` | プロダクションビルド（VSIX 用） |


### mobile-app (`packages/mobile-app`)

| コマンド | 内容 |
| --- | --- |
| `npm run sync` | web-app 静的ビルド + Capacitor sync |


## テスト


### ユニットテスト

```bash
# リポジトリルートで全パッケージのテストを実行
npx jest --no-coverage
```


### E2E テスト（Playwright）

Playwright ブラウザは Dockerfile のビルド時にインストール済み。

```bash
cd packages/web-app
npm run e2e
```

> 開発サーバーが起動していなくても、テスト内で自動起動される。

パッケージ更新でブラウザバージョンが変わった場合は再インストールする。

```bash
npx playwright install --with-deps
```


## VS Code 拡張機能


### デバッグ起動

1. VS Code でリポジトリを開く
2. `F5` で拡張機能のデバッグ起動
3. Extension Development Host で `.md` ファイルを開く
4. 右クリック → 「Open with Markdown Editor」を選択


### VSIX ファイルの作成

```bash
cd packages/vscode-extension
npx vsce package --no-dependencies
```

`anytime-markdown-<version>.vsix` が生成される。


### ローカルへのインストール

```bash
code --install-extension anytime-markdown-<version>.vsix
```

または VS Code のコマンドパレット → 「Extensions: Install from VSIX...」で指定する。


### Marketplace への公開

```bash
cd packages/vscode-extension
npx vsce publish --no-dependencies --pat <your-token>
```


## Android アプリ

Web アプリを Capacitor でラップした Android アプリ。\
Dev Container 内では `npm run sync` まで実行できるが、Android Studio の起動やエミュレータはホスト側（Windows / Mac）で行う。


### ビルド手順

```bash
# リポジトリルートで依存パッケージをインストール
npm install

# 静的ビルド + Capacitor sync
cd packages/mobile-app
npm run sync
```


### APK ビルド（WSL 内）

JDK 21 が必要。

```bash
sudo apt install -y openjdk-21-jdk
cd packages/mobile-app/android
./gradlew assembleDebug
```

出力先: `app/build/outputs/apk/debug/app-debug.apk`


### エミュレータで確認（Windows 側）

1. Android Studio → Device Manager → Virtual Device を作成（API 35 推奨）
2. エミュレータを起動
3. `\\wsl$\` 経由で APK ファイルをエミュレータにドラッグ＆ドロップ


### リリースビルド

```bash
cd packages/mobile-app/android

# キーストアファイルと keystore.properties の存在を確認
ls anytime-markdown-release.keystore
cat keystore.properties

# AAB を生成
./gradlew bundleRelease
```

出力先: `app/build/outputs/bundle/release/app-release.aab`


## CI（GitHub Actions）

| ワークフロー | トリガー | 内容 |
| --- | --- | --- |
| `publish-vscode-extension.yml` | PR + push（master） | テスト・ビルド・VSIX 公開 |
| `daily-build.yml` | 毎日 JST 6:00 | フルビルド + CodeQL + SonarCloud |

CI のステップ: `npm ci` → `npm audit` → `tsc --noEmit` → `lint` → ユニットテスト → SonarCloud → E2E → `next build` → VSIX パッケージング


## トラブルシューティング

**`node_modules` の権限エラー**\
Dev Container 初回起動後に `npm install` が失敗する場合、`node_modules` の所有権を確認する。

```bash
sudo chown -R node:node /anytime-markdown/node_modules
npm install
```

**Playwright ブラウザが見つからない**\
パッケージ更新後にブラウザバージョンが変わった場合に発生する。

```bash
npx playwright install --with-deps
```

**`anytime-markdown-docs` のマウント失敗**\
`docker-compose.yml` がホスト側の `~/anytime-markdown-docs/` をマウントする。\
ディレクトリが存在しない場合、コンテナ起動前に作成しておく。

```bash
mkdir -p ~/anytime-markdown-docs
```

**Android Studio から APK が見えない**\
WSL2 上のファイルには `\\wsl$\Ubuntu\home\<user>\...` でアクセスする。\
`\\wsl$\` が表示されない場合は、エクスプローラーのアドレスバーに直接パスを入力する。
