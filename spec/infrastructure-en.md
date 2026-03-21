# Infrastructure

Updated: 2026-03-21

Infrastructure documentation for Anytime Markdown, organized with diagrams and tables.
Covers overall architecture, CI/CD, data flow, development environment, and S3 folder structure.


## 1. Overall Architecture

Available from 3 platforms: browser, VS Code, and Android.
Hosted on Netlify with document storage on AWS S3.

```mermaid
flowchart TB
    %% Style definitions
    classDef user fill:#e3f2fd,stroke:#1565c0,color:#0d47a1
    classDef hosting fill:#e8f5e9,stroke:#2e7d32,color:#1b5e20
    classDef storage fill:#fff3e0,stroke:#ef6c00,color:#e65100
    classDef cicd fill:#f3e5f5,stroke:#7b1fa2,color:#4a148c
    classDef external fill:#fce4ec,stroke:#c62828,color:#b71c1c

    %% Node definitions
    Browser["Browser<br/><small>Chrome / Edge / Firefox</small>"]
    VSCode["VS Code<br/><small>Runs as extension</small>"]
    Android["Android Device<br/><small>Capacitor wrapper</small>"]

    Netlify["Netlify<br/><small>Next.js 15 hosting</small>"]
    S3["AWS S3<br/><small>ap-northeast-1</small>"]
    Marketplace["VS Code Marketplace<br/><small>Extension distribution</small>"]

    GHA["GitHub Actions<br/><small>CI/CD pipeline</small>"]
    GitHub["GitHub<br/><small>Source code management</small>"]

    GA["Google Analytics<br/><small>Access analytics</small>"]
    PlantUML["PlantUML Server<br/><small>Diagram rendering</small>"]

    %% Connection definitions
    Browser ==> Netlify
    VSCode -.-> Marketplace
    Android -.-> Netlify

    Netlify --> S3
    Netlify -.-> GA
    Netlify -.-> PlantUML

    GitHub ==> GHA
    GHA ==> Netlify
    GHA ==> Marketplace

    %% Style application
    class Browser,VSCode,Android user
    class Netlify hosting
    class S3 storage
    class GHA,GitHub cicd
    class GA,PlantUML,Marketplace external
```

| Color | Category | Targets |
| --- | --- | --- |
| Blue | User / Client | Browser, VS Code, Android device |
| Green | Hosting | Netlify |
| Orange | Storage | AWS S3 |
| Purple | CI/CD | GitHub, GitHub Actions |
| Red | External services | Google Analytics, PlantUML, Marketplace |


## 2. CI/CD Pipeline

Automated pipeline via GitHub Actions. Runs build, test, and deploy on push, plus scheduled security audits.

```mermaid
flowchart LR
    %% Style definitions
    classDef trigger fill:#e3f2fd,stroke:#1565c0,color:#0d47a1
    classDef step fill:#f3e5f5,stroke:#7b1fa2,color:#4a148c
    classDef output fill:#e8f5e9,stroke:#2e7d32,color:#1b5e20
    classDef schedule fill:#fff3e0,stroke:#ef6c00,color:#e65100

    %% Node definitions
    Push(["push to master/develop"])
    Daily(["Daily JST 6:00"])
    Weekly(["Weekly Sun JST 6:00"])

    Audit["Security Audit<br/><small>npm audit</small>"]
    TypeCheck["Type Check<br/><small>tsc --noEmit</small>"]
    Lint["Static Analysis<br/><small>ESLint</small>"]
    UnitTest["Unit Tests<br/><small>Jest</small>"]
    E2E["E2E Tests<br/><small>Playwright</small>"]
    Build["Build<br/><small>Next.js + Webpack</small>"]

    DeployWeb["Netlify Deploy"]
    PublishExt["Marketplace Publish<br/><small>vsce publish</small>"]
    VSIX["VSIX Artifact<br/><small>GitHub Artifacts</small>"]
    CacheClean["Cache Cleanup<br/><small>Delete entries older than 7 days</small>"]

    %% Connection definitions
    Push ==> Audit --> TypeCheck --> Lint --> UnitTest --> E2E --> Build
    Build ==> DeployWeb
    Build ==> PublishExt
    Build --> VSIX

    Daily ==> Audit
    Weekly ==> CacheClean

    %% Style application
    class Push,Daily,Weekly trigger
    class Audit,TypeCheck,Lint,UnitTest,E2E,Build step
    class DeployWeb,PublishExt,VSIX output
    class CacheClean schedule
```

| Color | Category | Targets |
| --- | --- | --- |
| Blue | Trigger | push, scheduled runs |
| Purple | Step | Audit, type check, lint, tests, build |
| Green | Artifact / Deploy | Netlify deploy, Marketplace publish, VSIX |
| Orange | Schedule | Cache cleanup |


## 3. Data Flow (Document Management)

Data flow for landing page document display and CMS operations (upload, delete, layout management).
CMS operations are protected by Basic authentication.

```mermaid
flowchart LR
    %% Style definitions
    classDef client fill:#e3f2fd,stroke:#1565c0,color:#0d47a1
    classDef api fill:#f3e5f5,stroke:#7b1fa2,color:#4a148c
    classDef storage fill:#fff3e0,stroke:#ef6c00,color:#e65100
    classDef auth fill:#fce4ec,stroke:#c62828,color:#b71c1c

    %% Node definitions
    WebApp["Web App<br/><small>Browser</small>"]
    BasicAuth{"Basic Auth<br/><small>For CMS operations</small>"}

    ListAPI["GET /api/docs<br/><small>List documents</small>"]
    ReadAPI["GET /api/docs/content<br/><small>Get document</small>"]
    UploadAPI["POST /api/docs/upload<br/><small>Upload md & images</small>"]
    DeleteAPI["DELETE /api/docs<br/><small>Delete file</small>"]
    LayoutAPI["PUT/GET /api/sites/layout<br/><small>Layout management</small>"]

    Bucket["S3 Bucket<br/><small>docs/ prefix</small>"]
    LayoutJSON["_layout.json<br/><small>Site structure definition</small>"]

    %% Connection definitions
    WebApp --> ListAPI & ReadAPI
    WebApp --> BasicAuth
    BasicAuth --> UploadAPI & DeleteAPI & LayoutAPI

    ListAPI & ReadAPI --> Bucket
    UploadAPI & DeleteAPI --> Bucket
    LayoutAPI --> LayoutJSON

    LayoutJSON -.-> Bucket

    %% Style application
    class WebApp client
    class ListAPI,ReadAPI,UploadAPI,DeleteAPI,LayoutAPI api
    class Bucket,LayoutJSON storage
    class BasicAuth auth
```

| Color | Category | Targets |
| --- | --- | --- |
| Blue | Client | Web app |
| Purple | API Endpoint | List, get, upload, delete, layout |
| Orange | Storage | S3 bucket, `_layout.json` |
| Red | Auth | Basic auth |


## 4. Development Environment

Development runs on a Docker Dev Container. Uses Node.js monorepo with editor-core as a shared library across platforms.

```mermaid
flowchart TB
    %% Style definitions
    classDef container fill:#e0f2f1,stroke:#00695c,color:#004d40
    classDef tool fill:#f3e5f5,stroke:#7b1fa2,color:#4a148c
    classDef runtime fill:#e3f2fd,stroke:#1565c0,color:#0d47a1

    %% Node definitions
    Docker["Docker Container<br/><small>Node.js 24-slim base</small>"]

    subgraph DevContainer["Dev Container"]
        Node["Node.js 24<br/><small>npm workspaces</small>"]
        Python["Python / uv<br/><small>Serena MCP Server</small>"]
        Playwright["Playwright<br/><small>E2E test browsers</small>"]
        ClaudeCLI["Claude Code CLI<br/><small>AI assistant</small>"]
    end

    subgraph Packages["Monorepo Structure"]
        Core["editor-core<br/><small>Shared editor library</small>"]
        Web["web-app<br/><small>Next.js 15 PWA</small>"]
        Ext["vscode-extension<br/><small>Webview extension</small>"]
        Mobile["mobile-app<br/><small>Capacitor + Android</small>"]
    end

    Port["localhost:3000<br/><small>Dev server</small>"]

    %% Connection definitions
    Docker ==> DevContainer
    DevContainer --> Packages
    Core --> Web & Ext & Mobile
    Web --> Port

    %% Style application
    class Docker container
    class Node,Python,Playwright,ClaudeCLI tool
    class Core,Web,Ext,Mobile,Port runtime
```

| Color | Category | Targets |
| --- | --- | --- |
| Teal | Container | Docker |
| Purple | Tools | Node.js, Python, Playwright, Claude Code CLI |
| Blue | Runtime / Packages | editor-core, web-app, vscode-extension, mobile-app, dev server |


## 5. Component List

Detailed specifications for each service and tool.

### 5.1 Hosting & Storage

| Service | Purpose | Region |
| --- | --- | --- |
| Netlify | Web app hosting | Auto |
| AWS S3 | Document storage | `ap-northeast-1` |
| VS Code Marketplace | Extension distribution | - |
| GitHub | Source code management / CI/CD | - |


### 5.2 S3 Folder Structure

Topic-based folders with language-specific md files and shared images.

```
s3://{S3_DOCS_BUCKET}/
└── {S3_DOCS_PREFIX}/              # Default: "docs/"
    ├── _layout.json               # Site structure definition (categories & items)
    ├── getting-started/
    │   ├── index-ja.md            # Japanese document
    │   ├── index-en.md            # English document
    │   └── images/                # Images shared across languages
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

| Item | Convention |
| --- | --- |
| Topic folder | `{topic}/` — one folder per topic |
| Japanese md | `index-ja.md` |
| English md | `index-en.md` |
| Image folder | `{topic}/images/` — shared across languages |
| Language-specific images | Named as `{topic}/images/{name}-ja.png` |
| Image references in md | Relative path `![description](./images/screenshot.png)` |
| `_layout.json` key | `docs/features/index-ja.md` format |
| URL parameter | `?key=docs/features/index-ja.md` |

> Relative image paths are automatically converted to CloudFront URLs (or API URLs) when `/api/docs/content` returns the document.


### 5.3 External Services

| Service | Purpose | Required |
| --- | --- | :---: |
| Google Analytics | Access analytics | x |
| PlantUML Server | Diagram rendering | x |


### 5.4 Auth & Security

| Target | Method | Environment Variables |
| --- | --- | --- |
| CMS operations (S3) | Basic auth | `CMS_BASIC_USER` / `CMS_BASIC_PASSWORD` |
| Marketplace publish | Personal Access Token | `VSCE_PAT` (GitHub Secrets) |
| AWS S3 access | IAM access key | `ANYTIME_AWS_ACCESS_KEY_ID` / `ANYTIME_AWS_SECRET_ACCESS_KEY` |


### 5.5 Build Artifacts

| Artifact | Output Path | Distribution |
| --- | --- | --- |
| Web app | `.next/` | Netlify |
| Static HTML (mobile) | `web-app/out/` | Capacitor |
| VSIX package | `vscode-extension/*.vsix` | Marketplace / GitHub Artifacts |
| Android APK/AAB | `mobile-app/android/app/build/outputs/` | Play Store (manual) |
