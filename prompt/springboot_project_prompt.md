
# Spring Boot バックエンドプロジェクト生成プロンプト（ChatGPT用）

以下の要件に従って、Java Spring Boot を用いたバックエンドプロジェクトを構築してください。

---

## 🔧 プロジェクトの概要

- REST API を持つ Spring Boot アプリケーションを作成してください。
- 主な機能はユーザー登録、ログイン、JWT 認証です。
- アーキテクチャは Controller - Service - Repository - Entity 層で構成。
- セキュリティは Spring Security + JWT を用いて構築してください。

---

## 🏗 プロジェクト構成

### 1. エンティティ層（Entity）
- `User`: id, username, email, password, roles（@ManyToMany）
- `Role`: id, name（例: ROLE_USER, ROLE_ADMIN）

### 2. リポジトリ層（Repository）
- `UserRepository`: ユーザー名やメールで検索できるメソッドを追加
- `RoleRepository`: 役割名で検索できるメソッドを追加

### 3. サービス層（Service）
- `UserService`
  - `registerUser(UserRegisterRequest)`
  - `getUserById(Long id)`
- `AuthService`
  - `authenticate(LoginRequest)`（ログイン → JWT発行）

### 4. コントローラー層（Controller）
- `AuthController`（`/api/auth`）
  - `POST /login`
  - `POST /register`
- `UserController`（`/api/users`）
  - `GET /{id}`（認証必須）

---

## 🔐 セキュリティ構成

### JWT & Spring Security
- `UserDetailsServiceImpl`: Spring Security の `UserDetailsService` 実装
- `JwtTokenUtil`: JWT の発行・検証ユーティリティ
- `JwtAuthFilter`: リクエストごとに JWT を検証するフィルター
- `AuthEntryPointJwt`: 未認証アクセス時のレスポンス処理
- `WebSecurityConfig`: `SecurityFilterChain`、パス制御などの設定

---

## 📨 DTO / レスポンスモデル

- `UserRegisterRequest`: username, email, password（@Valid）
- `LoginRequest`: username/email, password
- `AuthResponse`: JWTトークンなど
- `UserResponse`: ユーザー情報を外部に返す用

---

## ⚙ 設定ファイル

### `application.properties`
- データベース（H2 推奨）
- JPA 設定（`hibernate.ddl-auto=update`）
- JWT 設定（シークレット、有効期限など）

### `pom.xml`
- spring-boot-starter-web
- spring-boot-starter-data-jpa
- spring-boot-starter-security
- spring-boot-starter-validation
- H2 データベース
- jjwt ライブラリ
- lombok（任意）

---

## 🛠 その他

### グローバル例外ハンドラ
- `GlobalExceptionHandler`（`@ControllerAdvice`）
  - `UserNotFoundException`, `MethodArgumentNotValidException` などをハンドリング

### カスタム例外クラス
- `UserNotFoundException`
- `InvalidCredentialsException` など

---

## ✅ 指示方法例（ChatGPTへ）

> このようなプロジェクトを構築してください：
> - Spring Boot 3.x
> - User登録 & JWTログイン機能
> - Controller, Service, Repository, Entity 層に分けて実装
> - セキュリティにSpring Security + JWT使用
> - 各種DTOとバリデーション対応
> - 設定ファイルとpom.xmlも含めてください

---
