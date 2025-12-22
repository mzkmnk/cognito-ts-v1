# 07 - ソーシャルログインの統合

## 学習の目的

- OAuth 2.0 / OpenID Connect の基本を理解する
- Google ログインを Cognito に統合する
- Hosted UI と カスタム UI の違いを学ぶ
- フェデレーションユーザーの管理方法を理解する

## 背景知識

### OAuth 2.0 / OIDC フロー

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    OAuth 2.0 Authorization Code Flow                     │
│                                                                          │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐             │
│  │ 1.開始  │───▶│ 2.認可  │───▶│ 3.コード│───▶│ 4.トークン│             │
│  │         │    │ リクエスト│    │ 取得    │    │ 交換    │             │
│  └─────────┘    └─────────┘    └─────────┘    └─────────┘             │
│       │              │              │              │                    │
│       ▼              ▼              ▼              ▼                    │
│  ユーザーが      Google の     ユーザーが     Cognito が              │
│  「Google で    ログイン画面   承認すると     コードを                │
│  ログイン」を   にリダイレクト  コードが発行   トークンに交換          │
│  クリック                                                               │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Hosted UI vs カスタム UI

| 方式        | 特徴                       | ユースケース                     |
| ----------- | -------------------------- | -------------------------------- |
| Hosted UI   | Cognito が提供する認証画面 | 素早い実装、カスタマイズ不要     |
| カスタム UI | 自前の認証画面             | ブランド統一、完全なカスタマイズ |

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Hosted UI                                         │
│                                                                          │
│  メリット:                                                               │
│  • 実装が簡単（設定のみ）                                                │
│  • セキュリティが担保されている                                          │
│  • ソーシャルログインボタンが自動生成                                    │
│                                                                          │
│  デメリット:                                                             │
│  • カスタマイズに制限がある                                              │
│  • Cognito ドメインへのリダイレクトが発生                                │
│  • ブランドの一貫性を保ちにくい                                          │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                        カスタム UI                                       │
│                                                                          │
│  メリット:                                                               │
│  • 完全なデザインコントロール                                            │
│  • シームレスな UX                                                       │
│  • ブランドの一貫性                                                      │
│                                                                          │
│  デメリット:                                                             │
│  • 実装が複雑                                                            │
│  • セキュリティ考慮が必要                                                │
│  • メンテナンスコストが高い                                              │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

## 概念の説明

### Google Cloud Console での設定

1. Google Cloud Console で OAuth 2.0 クライアントを作成
2. 承認済みリダイレクト URI に Cognito のコールバック URL を設定
3. クライアント ID とシークレットを取得

### Cognito での IdP 設定

```typescript
// CDK での Google IdP 設定
const googleProvider = new cognito.UserPoolIdentityProviderGoogle(
  this,
  "GoogleProvider",
  {
    userPool: this.userPool,
    clientId: "your-google-client-id",
    clientSecretValue: cdk.SecretValue.secretsManager("google-client-secret"),
    scopes: ["openid", "email", "profile"],
    attributeMapping: {
      email: cognito.ProviderAttribute.GOOGLE_EMAIL,
      givenName: cognito.ProviderAttribute.GOOGLE_GIVEN_NAME,
      familyName: cognito.ProviderAttribute.GOOGLE_FAMILY_NAME,
      profilePicture: cognito.ProviderAttribute.GOOGLE_PICTURE,
    },
  }
);
```

## コードサンプル

### CDK での完全な設定

```typescript
// infra/lib/cognito-stack.ts（ソーシャルログイン設定を追加）
import * as cdk from "aws-cdk-lib";
import * as cognito from "aws-cdk-lib/aws-cognito";
import * as secretsmanager from "aws-cdk-lib/aws-secretsmanager";
import { Construct } from "constructs";

interface CognitoStackProps extends cdk.StackProps {
  domainPrefix: string;
  callbackUrls: string[];
  logoutUrls: string[];
}

export class CognitoStack extends cdk.Stack {
  public readonly userPool: cognito.UserPool;
  public readonly userPoolClient: cognito.UserPoolClient;
  public readonly userPoolDomain: cognito.UserPoolDomain;

  constructor(scope: Construct, id: string, props: CognitoStackProps) {
    super(scope, id, props);

    // User Pool
    this.userPool = new cognito.UserPool(this, "TaskFlowUserPool", {
      userPoolName: "taskflow-user-pool",
      signInAliases: { email: true },
      selfSignUpEnabled: true,
      autoVerify: { email: true },
      standardAttributes: {
        email: { required: true, mutable: true },
        givenName: { required: false, mutable: true },
        familyName: { required: false, mutable: true },
      },
      customAttributes: {
        organizationId: new cognito.StringAttribute({ mutable: true }),
      },
    });

    // Cognito Domain（Hosted UI 用）
    this.userPoolDomain = this.userPool.addDomain("Domain", {
      cognitoDomain: {
        domainPrefix: props.domainPrefix, // 例: 'taskflow-auth'
      },
    });

    // Google Identity Provider
    const googleClientSecret = secretsmanager.Secret.fromSecretNameV2(
      this,
      "GoogleClientSecret",
      "taskflow/google-oauth-secret"
    );

    const googleProvider = new cognito.UserPoolIdentityProviderGoogle(
      this,
      "GoogleProvider",
      {
        userPool: this.userPool,
        clientId: process.env.GOOGLE_CLIENT_ID || "your-google-client-id",
        clientSecretValue: googleClientSecret.secretValue,
        scopes: ["openid", "email", "profile"],
        attributeMapping: {
          email: cognito.ProviderAttribute.GOOGLE_EMAIL,
          givenName: cognito.ProviderAttribute.GOOGLE_GIVEN_NAME,
          familyName: cognito.ProviderAttribute.GOOGLE_FAMILY_NAME,
          profilePicture: cognito.ProviderAttribute.GOOGLE_PICTURE,
        },
      }
    );

    // App Client（OAuth 設定付き）
    this.userPoolClient = this.userPool.addClient("TaskFlowWebClient", {
      userPoolClientName: "taskflow-web-client",
      authFlows: {
        userPassword: true,
        userSrp: true,
        custom: true,
      },
      oAuth: {
        flows: {
          authorizationCodeGrant: true,
          implicitCodeGrant: false, // セキュリティ上、無効化推奨
        },
        scopes: [
          cognito.OAuthScope.OPENID,
          cognito.OAuthScope.EMAIL,
          cognito.OAuthScope.PROFILE,
        ],
        callbackUrls: props.callbackUrls,
        logoutUrls: props.logoutUrls,
      },
      supportedIdentityProviders: [
        cognito.UserPoolClientIdentityProvider.COGNITO,
        cognito.UserPoolClientIdentityProvider.GOOGLE,
      ],
      generateSecret: false,
    });

    // Google Provider への依存関係を追加
    this.userPoolClient.node.addDependency(googleProvider);

    // 出力
    new cdk.CfnOutput(this, "HostedUIUrl", {
      value: `https://${props.domainPrefix}.auth.${this.region}.amazoncognito.com`,
      description: "Cognito Hosted UI URL",
    });
  }
}
```

### Amplify 設定の更新

```typescript
// apps/web/src/app/auth/amplify-config.ts
import { Amplify } from "aws-amplify";
import { environment } from "../../environments/environment";

export function configureAmplify(): void {
  Amplify.configure({
    Auth: {
      Cognito: {
        userPoolId: environment.cognito.userPoolId,
        userPoolClientId: environment.cognito.userPoolClientId,
        signUpVerificationMethod: "code",
        loginWith: {
          oauth: {
            domain: environment.cognito.domain, // 例: 'taskflow-auth.auth.ap-northeast-1.amazoncognito.com'
            scopes: ["openid", "email", "profile"],
            redirectSignIn: [environment.cognito.redirectSignIn],
            redirectSignOut: [environment.cognito.redirectSignOut],
            responseType: "code",
            providers: ["Google"],
          },
        },
      },
    },
  });
}
```

```typescript
// apps/web/src/environments/environment.ts
export const environment = {
  production: false,
  cognito: {
    userPoolId: "ap-northeast-1_xxxxxxxx",
    userPoolClientId: "xxxxxxxxxxxxxxxxxxxxxxxxxx",
    domain: "taskflow-auth.auth.ap-northeast-1.amazoncognito.com",
    redirectSignIn: "http://localhost:4200/auth/callback",
    redirectSignOut: "http://localhost:4200/",
  },
  apiUrl: "http://localhost:3000",
};
```

### AuthService へのソーシャルログインメソッド追加

```typescript
// apps/web/src/app/auth/services/auth.service.ts に追加
import { signInWithRedirect, signOut } from 'aws-amplify/auth';

// AuthService クラスに追加

/**
 * Google でサインイン（Hosted UI にリダイレクト）
 */
async signInWithGoogle(): Promise<void> {
  this._isLoading.set(true);
  this._error.set(null);

  try {
    await signInWithRedirect({ provider: 'Google' });
  } catch (error) {
    const message = error instanceof Error ? error.message : 'Google ログインに失敗しました';
    this._error.set(message);
    throw error;
  } finally {
    this._isLoading.set(false);
  }
}

/**
 * フェデレーションサインアウト
 */
async federatedSignOut(): Promise<void> {
  this._isLoading.set(true);
  this._error.set(null);

  try {
    await signOut({ global: true });
    this._user.set(null);
  } catch (error) {
    const message = error instanceof Error ? error.message : 'ログアウトに失敗しました';
    this._error.set(message);
    throw error;
  } finally {
    this._isLoading.set(false);
  }
}
```

### OAuth コールバックコンポーネント

```typescript
// apps/web/src/app/auth/components/oauth-callback/oauth-callback.component.ts
import { Component, inject, OnInit } from "@angular/core";
import { CommonModule } from "@angular/common";
import { Router } from "@angular/router";
import { Hub } from "aws-amplify/utils";
import { AuthService } from "../../services/auth.service";

@Component({
  selector: "app-oauth-callback",
  standalone: true,
  imports: [CommonModule],
  template: `
    <div class="callback-container">
      <div class="loading">
        <p>認証処理中...</p>
        <div class="spinner"></div>
      </div>
    </div>
  `,
  styles: [
    `
      .callback-container {
        display: flex;
        justify-content: center;
        align-items: center;
        height: 100vh;
      }
      .loading {
        text-align: center;
      }
      .spinner {
        width: 40px;
        height: 40px;
        border: 4px solid #f3f3f3;
        border-top: 4px solid #3498db;
        border-radius: 50%;
        animation: spin 1s linear infinite;
        margin: 1rem auto;
      }
      @keyframes spin {
        0% {
          transform: rotate(0deg);
        }
        100% {
          transform: rotate(360deg);
        }
      }
    `,
  ],
})
export class OAuthCallbackComponent implements OnInit {
  private readonly authService = inject(AuthService);
  private readonly router = inject(Router);

  ngOnInit(): void {
    // Amplify Hub でサインインイベントをリッスン
    Hub.listen("auth", async ({ payload }) => {
      switch (payload.event) {
        case "signInWithRedirect":
          // サインイン成功
          await this.authService.checkAuthState();
          this.router.navigate(["/dashboard"]);
          break;
        case "signInWithRedirect_failure":
          // サインイン失敗
          console.error("OAuth sign-in failed:", payload.data);
          this.router.navigate(["/auth/sign-in"], {
            queryParams: { error: "oauth_failed" },
          });
          break;
      }
    });
  }
}
```

### サインインコンポーネントの更新

```typescript
// apps/web/src/app/auth/components/sign-in/sign-in.component.ts
// ソーシャルログインボタンを追加

template: `
  <div class="auth-container">
    <h1>ログイン</h1>

    <!-- ソーシャルログイン -->
    <div class="social-login">
      <button
        class="btn-google"
        (click)="signInWithGoogle()"
        [disabled]="authService.isLoading()"
      >
        <svg class="google-icon" viewBox="0 0 24 24">
          <path fill="#4285F4" d="M22.56 12.25c0-.78-.07-1.53-.2-2.25H12v4.26h5.92c-.26 1.37-1.04 2.53-2.21 3.31v2.77h3.57c2.08-1.92 3.28-4.74 3.28-8.09z"/>
          <path fill="#34A853" d="M12 23c2.97 0 5.46-.98 7.28-2.66l-3.57-2.77c-.98.66-2.23 1.06-3.71 1.06-2.86 0-5.29-1.93-6.16-4.53H2.18v2.84C3.99 20.53 7.7 23 12 23z"/>
          <path fill="#FBBC05" d="M5.84 14.09c-.22-.66-.35-1.36-.35-2.09s.13-1.43.35-2.09V7.07H2.18C1.43 8.55 1 10.22 1 12s.43 3.45 1.18 4.93l2.85-2.22.81-.62z"/>
          <path fill="#EA4335" d="M12 5.38c1.62 0 3.06.56 4.21 1.64l3.15-3.15C17.45 2.09 14.97 1 12 1 7.7 1 3.99 3.47 2.18 7.07l3.66 2.84c.87-2.6 3.3-4.53 6.16-4.53z"/>
        </svg>
        Google でログイン
      </button>
    </div>

    <div class="divider">
      <span>または</span>
    </div>

    <!-- 従来のログインフォーム -->
    <form [formGroup]="form" (ngSubmit)="onSubmit()">
      <!-- ... 既存のフォーム ... -->
    </form>
  </div>
`,

styles: [`
  .social-login {
    margin-bottom: 1.5rem;
  }
  .btn-google {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 100%;
    padding: 0.75rem;
    background: #fff;
    border: 1px solid #ddd;
    border-radius: 4px;
    cursor: pointer;
    font-size: 1rem;
  }
  .btn-google:hover {
    background: #f5f5f5;
  }
  .google-icon {
    width: 20px;
    height: 20px;
    margin-right: 0.5rem;
  }
  .divider {
    display: flex;
    align-items: center;
    margin: 1.5rem 0;
  }
  .divider::before,
  .divider::after {
    content: '';
    flex: 1;
    border-bottom: 1px solid #ddd;
  }
  .divider span {
    padding: 0 1rem;
    color: #666;
  }
`],

// コンポーネントクラスに追加
async signInWithGoogle(): Promise<void> {
  try {
    await this.authService.signInWithGoogle();
  } catch {
    // エラーは AuthService で処理済み
  }
}
```

## 実装タスク

### タスク 1: ルーティングの更新

OAuth コールバック用のルートを追加してください。

<details>
<summary>回答</summary>

```typescript
// apps/web/src/app/app.routes.ts に追加
{
  path: 'auth',
  children: [
    // ... 既存のルート ...
    {
      path: 'callback',
      loadComponent: () =>
        import('./auth/components/oauth-callback/oauth-callback.component').then(
          (m) => m.OAuthCallbackComponent
        ),
    },
  ],
},
```

</details>

## よくある間違い

### ❌ リダイレクト URI の不一致

```typescript
// 悪い例: 環境ごとに異なる URL を設定し忘れ
callbackUrls: ['http://localhost:4200/callback'],
// 本番環境で動作しない
```

### ✅ 環境ごとに適切な URL を設定

```typescript
// 良い例: 全環境の URL を設定
callbackUrls: [
  'http://localhost:4200/auth/callback',
  'https://taskflow.example.com/auth/callback',
],
```

## まとめ

この章では以下を学びました：

- OAuth 2.0 / OIDC の基本フロー
- Google ログインの Cognito への統合
- Hosted UI の設定と使用方法
- フェデレーションユーザーの認証フロー

## 確認クイズ

<details>
<summary>Q1: Authorization Code Flow と Implicit Flow の違いは？</summary>

**A1:**

- Authorization Code Flow: 認可コードを取得後、サーバーサイドでトークンに交換。より安全
- Implicit Flow: トークンが直接 URL フラグメントで返される。セキュリティリスクがあり非推奨

SPA では Authorization Code Flow + PKCE を使用することが推奨されています。

</details>

<details>
<summary>Q2: フェデレーションユーザーとローカルユーザーの違いは？</summary>

**A2:**

- フェデレーションユーザー: 外部 IdP（Google など）で認証。パスワードは Cognito に保存されない
- ローカルユーザー: Cognito で直接認証。パスワードは Cognito に保存される

同じメールアドレスで両方のユーザーが存在する場合、リンクして統合することも可能です。

</details>

---

前の章: [06 - パスキー認証の実装](./06-passkey-authentication.md)
次の章: [08 - グループと権限管理](./08-groups-permissions.md)
