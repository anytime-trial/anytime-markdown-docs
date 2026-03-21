# インフラ構成

更新日: 2026-03-21

Anytime Markdown のインフラ構成を図と表で整理したドキュメント。
全体構成、CI/CD、データフロー、開発環境、S3 フォルダ構成を含む。


## 1. 全体構成

ブラウザ・VS Code・Android の3プラットフォームから利用できる。
ホスティングは Netlify、ドキュメント格納は AWS S3 を使用する。

```mermaid
flowchart TB
    %% スタイル定義
    classDef user fill:#e3f2fd,stroke:#1565c0,color:#0d47a1
    classDef hosting fill:#e8f5e9,stroke:#2e7d32,color:#1b5e20
    classDef storage fill:#fff3e0,stroke:#ef6c00,color:#e65100
    classDef cicd fill:#f3e5f5,stroke:#7b1fa2,color:#4a148c
    classDef external fill:#fce4ec,stroke:#c62828,color:#b71c1c
    classDef dev fill:#e0f2f1,stroke:#00695c,color:#004d40

    %% ノード定義
    Browser["ブラウザ<br/><small>Chrome / Edge / Firefox</small>"]
    VSCode["VS Code<br/><small>拡張機能として動作</small>"]
    Android["Android 端末<br/><small>Capacitor ラッパー</small>"]

    Netlify["Netlify<br/><small>Next.js 15 ホスティング</small>"]
    S3["AWS S3<br/><small>ap-northeast-1</small>"]
    Marketplace["VS Code Marketplace<br/><small>拡張機能配布</small>"]

    GHA["GitHub Actions<br/><small>CI/CD パイプライン</small>"]
    GitHub["GitHub<br/><small>ソースコード管理</small>"]

    GA["Google Analytics<br/><small>アクセス解析</small>"]
    PlantUML["PlantUML Server<br/><small>図レンダリング</small>"]

    %% 接続定義
    Browser ==> Netlify
    VSCode -.-> Marketplace
    Android -.-> Netlify

    Netlify --> S3
    Netlify -.-> GA
    Netlify -.-> PlantUML

    GitHub ==> GHA
    GHA ==> Netlify
    GHA ==> Marketplace

    %% スタイル適用
    class Browser,VSCode,Android user
    class Netlify hosting
    class S3 storage
    class GHA,GitHub cicd
    class GA,PlantUML,Marketplace external
```

| 色 | 分類 | 対象 |
| --- | --- | --- |
| 青 | ユーザー / クライアント | ブラウザ、VS Code、Android 端末 |
| 緑 | ホスティング | Netlify |
| 橙 | ストレージ | AWS S3 |
| 紫 | CI/CD | GitHub、GitHub Actions |
| 赤 | 外部サービス | Google Analytics、PlantUML、Marketplace |


## 2. CI/CD パイプライン

GitHub Actions による自動化パイプライン。push 時のビルド・テスト・デプロイに加え、定期的なセキュリティ監査を実施する。

```mermaid
flowchart LR
    %% スタイル定義
    classDef trigger fill:#e3f2fd,stroke:#1565c0,color:#0d47a1
    classDef step fill:#f3e5f5,stroke:#7b1fa2,color:#4a148c
    classDef output fill:#e8f5e9,stroke:#2e7d32,color:#1b5e20
    classDef schedule fill:#fff3e0,stroke:#ef6c00,color:#e65100

    %% ノード定義
    Push(["push to master/develop"])
    Daily(["毎日 JST 6:00"])
    Weekly(["毎週日曜 JST 6:00"])

    Audit["セキュリティ監査<br/><small>npm audit</small>"]
    TypeCheck["型チェック<br/><small>tsc --noEmit</small>"]
    Lint["静的解析<br/><small>ESLint</small>"]
    UnitTest["ユニットテスト<br/><small>Jest</small>"]
    E2E["E2E テスト<br/><small>Playwright</small>"]
    Build["ビルド<br/><small>Next.js + Webpack</small>"]

    DeployWeb["Netlify デプロイ"]
    PublishExt["Marketplace 公開<br/><small>vsce publish</small>"]
    VSIX["VSIX アーティファクト<br/><small>GitHub Artifacts</small>"]
    CacheClean["キャッシュクリーンアップ<br/><small>7日超過分削除</small>"]

    %% 接続定義
    Push ==> Audit --> TypeCheck --> Lint --> UnitTest --> E2E --> Build
    Build ==> DeployWeb
    Build ==> PublishExt
    Build --> VSIX

    Daily ==> Audit
    Weekly ==> CacheClean

    %% スタイル適用
    class Push,Daily,Weekly trigger
    class Audit,TypeCheck,Lint,UnitTest,E2E,Build step
    class DeployWeb,PublishExt,VSIX output
    class CacheClean schedule
```

| 色 | 分類 | 対象 |
| --- | --- | --- |
| 青 | トリガー | push、定期実行 |
| 紫 | ステップ | 監査、型チェック、Lint、テスト、ビルド |
| 緑 | 成果物 / デプロイ | Netlify デプロイ、Marketplace 公開、VSIX |
| 橙 | スケジュール | キャッシュクリーンアップ |


## 3. データフロー（ドキュメント管理）

ランディングページのドキュメント表示・CMS 操作（アップロード・削除・レイアウト管理）のデータフロー。
CMS 操作は Basic 認証で保護される。

```mermaid
flowchart LR
    %% スタイル定義
    classDef client fill:#e3f2fd,stroke:#1565c0,color:#0d47a1
    classDef api fill:#f3e5f5,stroke:#7b1fa2,color:#4a148c
    classDef storage fill:#fff3e0,stroke:#ef6c00,color:#e65100
    classDef auth fill:#fce4ec,stroke:#c62828,color:#b71c1c

    %% ノード定義
    WebApp["Web アプリ<br/><small>ブラウザ</small>"]
    BasicAuth{"Basic 認証<br/><small>CMS 操作時</small>"}

    ListAPI["GET /api/docs<br/><small>一覧取得</small>"]
    ReadAPI["GET /api/docs/content<br/><small>文書取得</small>"]
    UploadAPI["POST /api/docs/upload<br/><small>md・画像アップロード</small>"]
    DeleteAPI["DELETE /api/docs<br/><small>ファイル削除</small>"]
    LayoutAPI["PUT/GET /api/sites/layout<br/><small>レイアウト管理</small>"]

    Bucket["S3 バケット<br/><small>docs/ プレフィックス</small>"]
    LayoutJSON["_layout.json<br/><small>サイト構造定義</small>"]

    %% 接続定義
    WebApp --> ListAPI & ReadAPI
    WebApp --> BasicAuth
    BasicAuth --> UploadAPI & DeleteAPI & LayoutAPI

    ListAPI & ReadAPI --> Bucket
    UploadAPI & DeleteAPI --> Bucket
    LayoutAPI --> LayoutJSON

    LayoutJSON -.-> Bucket

    %% スタイル適用
    class WebApp client
    class ListAPI,ReadAPI,UploadAPI,DeleteAPI,LayoutAPI api
    class Bucket,LayoutJSON storage
    class BasicAuth auth
```

| 色 | 分類 | 対象 |
| --- | --- | --- |
| 青 | クライアント | Web アプリ |
| 紫 | API エンドポイント | 一覧、取得、アップロード、削除、レイアウト |
| 橙 | ストレージ | S3 バケット、`_layout.json` |
| 赤 | 認証 | Basic 認証 |


## 4. 開発環境

Docker Dev Container 上で開発する。Node.js モノレポ構成で、editor-core を共有ライブラリとして各プラットフォームが利用する。

```mermaid
flowchart TB
    %% スタイル定義
    classDef container fill:#e0f2f1,stroke:#00695c,color:#004d40
    classDef tool fill:#f3e5f5,stroke:#7b1fa2,color:#4a148c
    classDef runtime fill:#e3f2fd,stroke:#1565c0,color:#0d47a1

    %% ノード定義
    Docker["Docker コンテナ<br/><small>Node.js 24-slim ベース</small>"]

    subgraph DevContainer["Dev Container"]
        Node["Node.js 24<br/><small>npm workspaces</small>"]
        Python["Python / uv<br/><small>Serena MCP Server</small>"]
        Playwright["Playwright<br/><small>E2E テスト用ブラウザ</small>"]
        ClaudeCLI["Claude Code CLI<br/><small>AI アシスタント</small>"]
    end

    subgraph Packages["モノレポ構成"]
        Core["editor-core<br/><small>共通エディタライブラリ</small>"]
        Web["web-app<br/><small>Next.js 15 PWA</small>"]
        Ext["vscode-extension<br/><small>Webview 拡張</small>"]
        Mobile["mobile-app<br/><small>Capacitor + Android</small>"]
    end

    Port["localhost:3000<br/><small>開発サーバー</small>"]

    %% 接続定義
    Docker ==> DevContainer
    DevContainer --> Packages
    Core --> Web & Ext & Mobile
    Web --> Port

    %% スタイル適用
    class Docker container
    class Node,Python,Playwright,ClaudeCLI tool
    class Core,Web,Ext,Mobile,Port runtime
```

| 色 | 分類 | 対象 |
| --- | --- | --- |
| 青緑 | コンテナ | Docker |
| 紫 | ツール | Node.js、Python、Playwright、Claude Code CLI |
| 青 | ランタイム / パッケージ | editor-core、web-app、vscode-extension、mobile-app、開発サーバー |


## 5. 構成要素一覧

各サービス・ツールの詳細仕様。

### 5.1 ホスティング・ストレージ

| サービス | 用途 | リージョン |
| --- | --- | --- |
| Netlify | Web アプリホスティング | 自動 |
| AWS S3 | ドキュメント格納 | `ap-northeast-1` |
| VS Code Marketplace | 拡張機能配布 | - |
| GitHub | ソースコード管理・CI/CD | - |


### 5.2 S3 フォルダ構成

トピック別フォルダ + 言語別 md + 画像共有の構成を採用。

```
s3://{S3_DOCS_BUCKET}/
└── {S3_DOCS_PREFIX}/              # デフォルト: "docs/"
    ├── _layout.json               # サイト構造定義（カテゴリ・アイテム）
    ├── getting-started/
    │   ├── index-ja.md            # 日本語ドキュメント
    │   ├── index-en.md            # 英語ドキュメント
    │   └── images/                # 画像は言語共通
    │       ├── install-step1.png
    │       └── install-step2.png
    ├── features/
    │   ├── index-ja.md
    │   ├── index-en.md
    │   └── images/
    │       ├── editor-preview.png
    │       └── split-view.gif
    └── infrastructure/
        ├── index-ja.md
        ├── index-en.md
        └── images/
            └── architecture.png
```

| 項目 | 規則 |
| --- | --- |
| トピックフォルダ | `{topic}/` — 1 トピックにつき 1 フォルダ |
| 日本語 md | `index-ja.md` |
| 英語 md | `index-en.md` |
| 画像フォルダ | `{topic}/images/` — 言語間で共有 |
| 言語固有の画像 | `{topic}/images/{name}-ja.png` のように命名で対応 |
| md 内の画像参照 | 相対パス `![説明](./images/screenshot.png)` |
| `_layout.json` の key | `docs/features/index-ja.md` 形式 |
| URL パラメータ | `?key=docs/features/index-ja.md` |

> 画像の相対パスは `/api/docs/content` が返却時に CloudFront URL（または API URL）に自動変換する。


### 5.3 外部サービス


| サービス | 用途 | 必須 |
| --- | --- | :---: |
| Google Analytics | アクセス解析 | x |
| PlantUML Server | 図のレンダリング | x |


### 5.4 認証・セキュリティ

| 対象 | 方式 | 環境変数 |
| --- | --- | --- |
| CMS 操作（S3） | Basic 認証 | `CMS_BASIC_USER` / `CMS_BASIC_PASSWORD` |
| Marketplace 公開 | Personal Access Token | `VSCE_PAT`（GitHub Secrets） |
| AWS S3 アクセス | IAM アクセスキー | `ANYTIME_AWS_ACCESS_KEY_ID` / `ANYTIME_AWS_SECRET_ACCESS_KEY` |


### 5.5 ビルド成果物

| 成果物 | 出力先 | 配布先 |
| --- | --- | --- |
| Web アプリ | `.next/` | Netlify |
| 静的 HTML（モバイル用） | `web-app/out/` | Capacitor |
| VSIX パッケージ | `vscode-extension/*.vsix` | Marketplace / GitHub Artifacts |
| Android APK/AAB | `mobile-app/android/app/build/outputs/` | Play Store（手動） |
