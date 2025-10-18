# Traefik Forward Auth アーキテクチャドキュメント

## 概要

Traefik Forward Authは、Traefikリバースプロキシと連携して動作するOAuth/OIDC認証ミドルウェアサービスです。Traefikの Forward Authentication 機能を利用して、バックエンドサービスへのアクセス前にユーザー認証を行います。

## アーキテクチャ概要

```mermaid
graph TB
    User[ユーザー]
    Traefik[Traefikリバースプロキシ]
    TFA[Traefik Forward Auth]
    Backend[バックエンドサービス]
    OAuth[OAuthプロバイダー<br/>Google/OIDC/Generic OAuth]

    User -->|1. リクエスト| Traefik
    Traefik -->|2. 認証確認| TFA
    TFA -->|3a. 未認証| OAuth
    OAuth -->|4. 認証| User
    User -->|5. コールバック| Traefik
    Traefik -->|6. トークン交換| TFA
    TFA -->|7. Cookieセット| User
    TFA -->|8. 認証OK| Traefik
    Traefik -->|9. アクセス許可| Backend
    Backend -->|10. レスポンス| User
```

## 主要コンポーネント

### 1. エントリーポイント (`cmd/main.go`)

```mermaid
graph LR
    A[main] --> B[設定読み込み]
    B --> C[ロガー初期化]
    C --> D[設定バリデーション]
    D --> E[サーバー構築]
    E --> F[HTTPサーバー起動]
```

**責務:**
- グローバル設定の初期化
- ロガーのセットアップ
- HTTPサーバーの起動 (デフォルト: ポート4181)

### 2. サーバー (`internal/server.go`)

**主要ハンドラー:**

| ハンドラー | パス | 責務 |
|---------|------|------|
| RootHandler | / | すべてのリクエストのエントリーポイント |
| AllowHandler | (ルールベース) | 認証なしでリクエストを許可 |
| AuthHandler | (ルールベース) | 認証を要求し、Cookieを検証 |
| AuthCallbackHandler | /_oauth | OAuthコールバックを処理 |
| LogoutHandler | /_oauth/logout | ログアウト処理 |

### 3. 認証フロー (`internal/auth.go`)

```mermaid
sequenceDiagram
    participant U as ユーザー
    participant T as Traefik
    participant TFA as Forward Auth
    participant P as OAuthプロバイダー

    U->>T: リクエスト
    T->>TFA: X-Forwarded-* ヘッダー付きで転送

    alt Cookie存在 & 有効
        TFA->>TFA: Cookie検証
        TFA->>TFA: メール検証 (whitelist/domain)
        TFA->>T: 200 OK + X-Forwarded-User
        T->>U: バックエンドへアクセス
    else Cookie無効/期限切れ
        TFA->>TFA: CSRF Cookie生成
        TFA->>U: 302 リダイレクト (OAuthログインURL)
        U->>P: 認証
        P->>U: 認証コード + state
        U->>TFA: コールバック
        TFA->>TFA: CSRF/state検証
        TFA->>P: コード交換 → トークン
        TFA->>P: ユーザー情報取得
        TFA->>U: Auth Cookie設定 + リダイレクト
    end
```

### 4. 設定管理 (`internal/config.go`)

**主要設定項目:**

```mermaid
mindmap
  root((Config))
    認証設定
      Secret (署名用シークレット)
      Lifetime (Cookieの有効期限)
      CookieName
      CookieDomains
      InsecureCookie
    プロバイダー設定
      Google
        ClientID
        ClientSecret
        Prompt
      OIDC
        IssuerURL
        ClientID
        ClientSecret
      GenericOAuth
        AuthURL
        TokenURL
        UserURL
        ClientID
        ClientSecret
        Scopes
    アクセス制御
      DefaultAction (auth/allow)
      DefaultProvider
      Domains (許可ドメイン)
      Whitelist (許可メールアドレス)
      Rules (カスタムルール)
    その他
      Port
      Path (コールバックパス)
      LogoutRedirect
      AuthHost
```

### 5. プロバイダー (`internal/provider/`)

**プロバイダーインターフェース:**

```go
type Provider interface {
    Name() string
    GetLoginURL(redirectURI, state string) string
    ExchangeCode(redirectURI, code string) (string, error)
    GetUser(token string) (User, error)
    Setup() error
}
```

**実装プロバイダー:**

```mermaid
classDiagram
    class Provider {
        <<interface>>
        +Name() string
        +GetLoginURL() string
        +ExchangeCode() string, error
        +GetUser() User, error
        +Setup() error
    }

    class Google {
        +ClientID
        +ClientSecret
        +Scope
        +Prompt
    }

    class OIDC {
        +IssuerURL
        +ClientID
        +ClientSecret
        +provider
        +verifier
    }

    class GenericOAuth {
        +AuthURL
        +TokenURL
        +UserURL
        +ClientID
        +ClientSecret
        +Scopes
        +TokenStyle
    }

    class OAuthProvider {
        +Resource
        +Config
        +OAuthGetLoginURL()
        +OAuthExchangeCode()
    }

    Provider <|.. Google
    Provider <|.. OIDC
    Provider <|.. GenericOAuth
    OAuthProvider <|-- OIDC
    OAuthProvider <|-- GenericOAuth
```

## 動作フロー詳細

### 認証リクエストフロー

```mermaid
flowchart TD
    A[リクエスト受信] --> B[RootHandler]
    B --> C[X-Forwarded-*ヘッダーから<br/>リクエスト情報を復元]
    C --> D{ルールマッチ?}

    D -->|allow| E[AllowHandler]
    E --> F[200 OK]

    D -->|auth| G[AuthHandler]
    G --> H{Cookie存在?}

    H -->|No| I[authRedirect]
    I --> J[Nonce生成]
    J --> K[CSRF Cookie設定]
    K --> L[OAuthログインURLへリダイレクト]

    H -->|Yes| M{Cookie有効?}
    M -->|No| I
    M -->|Yes| N{メールアドレス許可?}
    N -->|No| I
    N -->|Yes| O[X-Forwarded-Userヘッダー設定]
    O --> F
```

### OAuthコールバックフロー

```mermaid
flowchart TD
    A[コールバック受信] --> B[AuthCallbackHandler]
    B --> C{stateパラメータ有効?}
    C -->|No| D[401 Unauthorized]

    C -->|Yes| E{CSRF Cookie存在?}
    E -->|No| D

    E -->|Yes| F{CSRF Cookie検証<br/>state一致?}
    F -->|No| D

    F -->|Yes| G[プロバイダー取得]
    G --> H[CSRF Cookieクリア]
    H --> I[コード交換<br/>認可コード → トークン]

    I -->|失敗| J[503 Service Unavailable]

    I -->|成功| K[ユーザー情報取得]
    K -->|失敗| J

    K -->|成功| L[Auth Cookie生成]
    L --> M[元のURLへリダイレクト]
```

### Cookie検証フロー

```mermaid
flowchart TD
    A[Cookie検証開始] --> B[Cookie値を分割<br/>hash | expires | email]
    B --> C{フォーマット正しい?}

    C -->|No| D[エラー: Invalid cookie format]

    C -->|Yes| E[MAC署名をデコード]
    E --> F[期待される署名を生成<br/>HMAC-SHA256 secret, domain, email, expires]
    F --> G{署名一致?}

    G -->|No| H[エラー: Invalid cookie mac]

    G -->|Yes| I{有効期限内?}
    I -->|No| J[エラー: Cookie has expired]

    I -->|Yes| K[メールアドレス検証]
    K --> L{whitelistに存在?}
    L -->|Yes| M[認証成功]

    L -->|No| N{許可ドメイン?}
    N -->|Yes| M
    N -->|No| O[認証失敗]
```

## セキュリティ機能

### 1. Cookie署名

```mermaid
graph LR
    A[Secret] --> D[HMAC-SHA256]
    B[Cookie Domain] --> D
    C[Email] --> D
    E[Expires] --> D
    D --> F[Base64エンコード]
    F --> G[Cookie MAC]
```

**Cookie形式:** `MAC|有効期限(Unix timestamp)|メールアドレス`

### 2. CSRF対策

```mermaid
sequenceDiagram
    participant C as クライアント
    participant S as サーバー

    Note over S: 認証開始
    S->>S: 32バイトのNonce生成
    S->>C: CSRF Cookie設定 (_forward_auth_csrf_XXXXXX)
    S->>C: state = nonce:provider:redirect_url

    Note over C: OAuthフロー
    C->>S: コールバック (state含む)
    S->>S: CSRF Cookie検証
    S->>S: state内のnonceとCookieの値が一致するか確認
    S->>C: CSRF Cookieクリア
```

### 3. アクセス制御

**ルールベースの制御:**

```mermaid
flowchart TD
    A[リクエスト] --> B{ルールマッチング}
    B -->|Rule 1| C{Action?}
    B -->|Rule 2| D{Action?}
    B -->|Default| E{DefaultAction?}

    C -->|allow| F[即座に許可]
    C -->|auth| G[認証要求]

    D -->|allow| F
    D -->|auth| H[認証要求<br/>カスタムプロバイダー]

    E -->|allow| F
    E -->|auth| I[デフォルトプロバイダーで認証]

    G --> J{メールアドレス検証}
    H --> K{メールアドレス検証<br/>ルール固有のwhitelist/domain}
    I --> L{メールアドレス検証<br/>グローバルwhitelist/domain}
```

**メールアドレス検証ロジック:**

1. **Whitelist検証:** 完全一致でメールアドレスをチェック
2. **Domain検証:** `@`以降のドメインをチェック
3. **MatchWhitelistOrDomain:** どちらか一方でOK (無効の場合は両方必要)

## 設定例

### 基本設定

```ini
# 必須設定
secret = your-secret-key-here
port = 4181

# プロバイダー設定
default-provider = google

[providers.google]
client-id = your-google-client-id
client-secret = your-google-client-secret
prompt = select_account

# アクセス制御
domain = example.com
whitelist = admin@example.com,user@example.com
```

### ルールベース設定

```ini
secret = your-secret-key

# デフォルトは認証必須
default-action = auth
default-provider = google

# 公開パスは認証不要
rule.public.action = allow
rule.public.rule = PathPrefix(`/public`)

# 管理者パスはホワイトリスト限定
rule.admin.action = auth
rule.admin.rule = PathPrefix(`/admin`)
rule.admin.whitelist = admin@example.com
rule.admin.provider = google

# API用にOIDC認証
rule.api.action = auth
rule.api.rule = PathPrefix(`/api`)
rule.api.provider = oidc

[providers.oidc]
issuer-url = https://accounts.google.com
client-id = your-oidc-client-id
client-secret = your-oidc-client-secret
```

## Traefikとの統合

### Docker Compose例

```yaml
services:
  traefik:
    image: traefik:v2.x
    command:
      - "--providers.docker=true"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock

  traefik-forward-auth:
    image: thomseddon/traefik-forward-auth:latest
    environment:
      - SECRET=something-random
      - PROVIDERS_GOOGLE_CLIENT_ID=your-client-id
      - PROVIDERS_GOOGLE_CLIENT_SECRET=your-client-secret
      - DOMAIN=example.com
    labels:
      - "traefik.http.middlewares.traefik-forward-auth.forwardauth.address=http://traefik-forward-auth:4181"
      - "traefik.http.middlewares.traefik-forward-auth.forwardauth.authResponseHeaders=X-Forwarded-User"

  whoami:
    image: traefik/whoami
    labels:
      - "traefik.http.routers.whoami.rule=Host(`whoami.example.com`)"
      - "traefik.http.routers.whoami.middlewares=traefik-forward-auth@docker"
```

### リクエストフロー

```mermaid
sequenceDiagram
    participant U as ユーザー
    participant T as Traefik
    participant TFA as Forward Auth
    participant W as Whoami (Backend)

    U->>T: GET https://whoami.example.com/

    Note over T: Middlewareチェック
    T->>TFA: GET http://traefik-forward-auth:4181/
    Note over T: X-Forwarded-Method: GET<br/>X-Forwarded-Host: whoami.example.com<br/>X-Forwarded-Uri: /<br/>X-Forwarded-Proto: https

    alt 未認証
        TFA->>U: 302 https://accounts.google.com/...
        U->>Google: 認証
        Google->>U: 302 https://whoami.example.com/_oauth?code=...&state=...
        U->>T: GET /_oauth?code=...
        T->>TFA: コールバック処理
        TFA->>Google: トークン交換
        TFA->>U: Set-Cookie + 302 https://whoami.example.com/
    end

    U->>T: GET https://whoami.example.com/ (Cookie付き)
    T->>TFA: 認証確認
    TFA->>T: 200 OK + X-Forwarded-User: user@example.com
    T->>W: リクエスト転送 (X-Forwarded-User含む)
    W->>T: レスポンス
    T->>U: レスポンス
```

## 技術スタック

- **言語:** Go
- **主要ライブラリ:**
  - `github.com/sirupsen/logrus` - ロギング
  - `github.com/traefik/traefik/v2/pkg/muxer/http` - ルーティング
  - `golang.org/x/oauth2` - OAuth2クライアント
  - `github.com/coreos/go-oidc` - OIDC検証
  - `github.com/thomseddon/go-flags` - CLI引数パース

## まとめ

Traefik Forward Authは、以下の特徴を持つ軽量な認証ゲートウェイです:

✅ **マルチプロバイダー対応** - Google、OIDC、汎用OAuth2
✅ **柔軟なアクセス制御** - ルールベース、ドメイン、ホワイトリスト
✅ **セキュア** - HMAC署名Cookie、CSRF対策
✅ **Traefik統合** - Forward Authミドルウェアとしてシームレスに動作
✅ **軽量** - 単一バイナリ、低リソース使用量

このアーキテクチャにより、既存のアプリケーションに認証機能を追加する際、アプリケーション側のコード変更なしに認証レイヤーを挿入できます。
