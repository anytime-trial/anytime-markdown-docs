# Anytime Markdown - プロジェクト構成 UML

## パッケージ図

```mermaid
graph TB
    subgraph "anytime-markdown (npm workspaces monorepo)"
        EC["@anytime-markdown/editor-core<br/>共有エディタライブラリ"]
        WA["@anytime-markdown/web-app<br/>Next.js Web アプリ"]
        VE["@anytime-markdown/vscode-extension<br/>VS Code 拡張機能"]
    end

    WA -->|"workspace 依存"| EC
    VE -->|"workspace 依存"| EC

    subgraph "外部依存"
        Tiptap["@tiptap/core + 拡張"]
        React["React 19"]
        MUI["MUI 7"]
        Next["Next.js 15"]
        VSCodeAPI["VS Code Extension API"]
        Mermaid["Mermaid.js"]
        DOMPurify["DOMPurify"]
    end

    EC --> Tiptap
    EC --> React
    EC --> MUI
    WA --> Next
    VE --> VSCodeAPI
    EC --> Mermaid
    EC --> DOMPurify

```

## コンポーネント図

```mermaid
graph TB
    subgraph "editor-core"
        MEP["MarkdownEditorPage<br/>メイン UI ページ"]

        subgraph "Hooks"
            UME["useMarkdownEditor<br/>状態管理"]
            UES["useEditorSettings<br/>設定管理"]
            UFO["useEditorFileOps<br/>ファイル操作"]
            USM["useSourceMode<br/>ソースモード"]
            UMD["useMergeDiff<br/>マージ/Diff"]
            UOL["useOutline<br/>アウトライン"]
        end

        subgraph "Components"
            ET["EditorToolbar"]
            SB["StatusBar"]
            BM["EditorBubbleMenu"]
            SR["SearchReplaceBar"]
            OP["OutlinePanel"]
            MP["MergeEditorPanel"]
            ESP["EditorSettingsPanel"]
        end

        subgraph "Tiptap Extensions"
            BE["getBaseExtensions"]
            CI["CustomImage"]
            CT["CustomTable"]
            CB["CodeBlockWithMermaid"]
            DH["DiffHighlight"]
            HF["HeadingFoldExtension"]
            SRE["SearchReplaceExtension"]
        end

        subgraph "Utils"
            DE["diffEngine"]
            SH["sectionHelpers"]
            TH["tableHelpers"]
            SM["sanitizeMarkdown"]
        end

        MEP --> UME
        MEP --> UES
        MEP --> UFO
        MEP --> USM
        MEP --> UMD
        MEP --> UOL
        MEP --> ET
        MEP --> SB
        MEP --> BM
        MEP --> SR
        MEP --> OP
        MEP --> MP
        MEP --> ESP
        UME --> BE
    end

    subgraph "web-app"
        Page["page.tsx<br/>エントリーポイント"]
        Layout["layout.tsx"]
        Providers["providers.tsx<br/>MUI + next-intl"]

        Layout --> Providers
        Page -->|"dynamic import<br/>ssr: false"| MEP
    end

    subgraph "vscode-extension"
        EXT["extension.ts<br/>コマンド登録"]
        MEPR["MarkdownEditorProvider<br/>CustomTextEditorProvider"]
        APP["App.tsx<br/>Webview エントリー"]
        VAPI["vscodeApi.ts<br/>API ラッパー"]
        SHIMS["shims/<br/>next-intl, next-dynamic"]

        EXT --> MEPR
        APP -->|"dynamic import"| MEP
        APP --> VAPI
        APP --> SHIMS
    end

```

## クラス図（VS Code 拡張）

```mermaid
classDiagram
    class MarkdownEditorProvider {
        -static instance: MarkdownEditorProvider
        -activePanel: WebviewPanel
        -panels: Map~string, WebviewPanel~
        -readyPanels: Set~string~
        -readyResolvers: Map~string, Array~Function~~
        +compareFileUri: Uri
        +activeDocumentUri: Uri
        +static getInstance() MarkdownEditorProvider
        +static register(context) Disposable
        +postMessageToActivePanel(message) void
        +waitForReady(uri) Promise~void~
        +postMessageToPanel(uri, message) boolean
        +resolveCustomTextEditor(document, panel, token) Promise~void~
        -getHtmlForWebview(webview) string
    }

    class ExtensionCommands {
        +openEditor()
        +openEditorWithFile(uri)
        +compareWithMarkdownEditor(uri)
        +compareWithGitHead(resourceState)
    }

    MarkdownEditorProvider ..|> CustomTextEditorProvider
    ExtensionCommands --> MarkdownEditorProvider : uses

```

## シーケンス図（SCM Compare with Git HEAD）

```mermaid
sequenceDiagram
    actor User
    participant SCM as SCM パネル
    participant EXT as extension.ts
    participant Provider as MarkdownEditorProvider
    participant Git as Git API
    participant WV as Webview (React)

    User->>SCM: 右クリック → Compare with Git HEAD
    SCM->>EXT: compareWithGitHead(resourceState)
    EXT->>Provider: getInstance()
    EXT->>Git: getRepository(fileUri)
    EXT->>Git: show('HEAD', relativePath)
    Git-->>EXT: headContent

    EXT->>Provider: vscode.openWith(fileUri, viewType)

    alt 新規エディタ
        Provider->>Provider: resolveCustomTextEditor()
        Provider->>WV: HTML + Script ロード
        WV-->>Provider: ready メッセージ
        Provider->>WV: setBaseUri + setContent
        Provider->>Provider: readyPanels.add() + resolve waiters
    end

    EXT->>Provider: waitForReady(fileUri)
    Provider-->>EXT: resolved

    Note over EXT: 500ms 遅延（React 初期化待ち）

    EXT->>Provider: postMessageToPanel(loadCompareFile)
    Provider->>WV: loadCompareFile(headContent)
    WV->>WV: 比較モード ON + 右パネルに HEAD 表示

```

## 配置図（開発環境）

```mermaid
graph TD
    Browser["ブラウザ<br/>localhost:3001"]
    VSCode["VS Code<br/>Extension Development Host"]

    Browser -->|"HTTP :3001→:3000"| NextDev
    VSCode -->|"extensionHost"| VE_DIST

    subgraph "Docker Dev Container (node:24-alpine)"

        subgraph "ビルドプロセス"
            NextDev["Next.js Dev Server :3000"]
            Webpack["webpack --watch"]
        end

        subgraph "/anytime-markdown (bind mount)"
            EC_SRC["packages/editor-core/src<br/>共有エディタライブラリ"]
            WA_SRC["packages/web-app/src<br/>Next.js ソース"]
            VE_SRC["packages/vscode-extension/src<br/>Extension ソース"]
            VE_DIST["packages/vscode-extension/dist<br/>extension.js + webview.js"]
        end

        subgraph "ツール"
            Node["Node.js 24"]
            Git["Git"]
            ClaudeCLI["Claude Code CLI"]
        end

        NextDev -->|"トランスパイル"| WA_SRC
        NextDev -->|"参照"| EC_SRC
        Webpack -->|"バンドル"| VE_SRC
        Webpack -->|"参照"| EC_SRC
        Webpack -->|"出力"| VE_DIST

    end

    subgraph "ボリュームマウント"
        NM["node_modules (named volume)"]
        SSH["~/.ssh (cached mount)"]
        Claude["~/.claude (cached mount)"]
    end

    NM -.-> EC_SRC
    SSH -.-> Git
    Claude -.-> ClaudeCLI

```

### 開発ワークフロー

| 環境 | 起動方法 | ビルド | アクセス |
| --- | --- | --- | --- |
| Web アプリ | `cd packages/web-app && npm run dev` | Next.js (HMR) | localhost:3001 |
| VS Code 拡張 | F5 (Launch Extension) | webpack --watch | Extension Development Host |
| 型チェック | `npx tsc --noEmit` | TypeScript | \- |

### ポートマッピング

| ホスト | コンテナ | 用途 |
| --- | --- | --- |
| 3001 | 3000 | Next.js Dev Server |

## ディレクトリ構成

```
anytime-markdown/
├── package.json                          # workspaces 定義
├── tsconfig.base.json                    # 共通 TypeScript 設定
├── packages/
│   ├── editor-core/                      # 共有エディタライブラリ
│   │   ├── package.json                  # peer dependencies
│   │   └── src/
│   │       ├── index.ts                  # 全エクスポート
│   │       ├── MarkdownEditorPage.tsx    # メイン UI ページ
│   │       ├── useMarkdownEditor.ts      # 状態管理フック
│   │       ├── useEditorSettings.ts      # 設定コンテキスト
│   │       ├── editorExtensions.ts       # Tiptap 拡張セット
│   │       ├── components/               # UI コンポーネント
│   │       ├── extensions/               # Tiptap カスタム拡張
│   │       ├── hooks/                    # カスタムフック
│   │       ├── utils/                    # ユーティリティ
│   │       ├── constants/                # 定数・テンプレート
│   │       ├── i18n/                     # en.json, ja.json
│   │       └── __tests__/               # ユニットテスト
│   ├── web-app/                          # Next.js Web アプリ
│   │   ├── package.json
│   │   └── src/app/
│   │       ├── page.tsx                  # エントリー（動的ロード）
│   │       ├── layout.tsx                # メタデータ + プロバイダー
│   │       └── providers.tsx             # MUI + next-intl
│   └── vscode-extension/                 # VS Code 拡張機能
│       ├── package.json                  # commands, menus, customEditors
│       └── src/
│           ├── extension.ts              # コマンド登録
│           ├── providers/
│           │   └── MarkdownEditorProvider.ts  # Custom Editor 実装
│           └── webview/
│               ├── index.tsx             # Webview エントリー
│               ├── App.tsx               # localStorage ブリッジ
│               ├── vscodeApi.ts          # VS Code API ラッパー
│               └── shims/               # next-intl, next-dynamic polyfill

```