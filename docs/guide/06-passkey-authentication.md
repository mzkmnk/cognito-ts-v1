# 06 - パスキー認証の実装

## 学習の目的

- WebAuthn/FIDO2 の基本概念を理解する
- Cognito でのパスキー設定方法を学ぶ
- パスキーの登録フローを実装する
- パスキーによるサインインを実装する

## 背景知識

### パスキーとは

パスキー（Passkey）は、FIDO2/WebAuthn 標準に基づくパスワードレス認証方式です：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        パスキーの仕組み                                  │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    認証器（Authenticator）                       │   │
│  │                                                                   │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │   │
│  │  │ プラットフォーム │  │ ローミング  │  │ ハイブリッド          │ │   │
│  │  │ 認証器       │  │ 認証器      │  │                         │ │   │
│  │  │              │  │             │  │                         │ │   │
│  │  │ • Touch ID   │  │ • YubiKey   │  │ • スマートフォンを      │ │   │
│  │  │ • Face ID    │  │ • セキュリティ│  │   認証器として使用     │ │   │
│  │  │ • Windows    │  │   キー      │  │                         │ │   │
│  │  │   Hello      │  │             │  │                         │ │   │
│  │  └─────────────┘  └─────────────┘  └─────────────────────────┘ │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  特徴:                                                                   │
│  • パスワード不要                                                        │
│  • フィッシング耐性                                                      │
│  • 公開鍵暗号方式（秘密鍵はデバイスに保存）                              │
│  • 生体認証との組み合わせ                                                │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### WebAuthn の登録フロー

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     パスキー登録フロー                                   │
│                                                                          │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐             │
│  │ 1.登録  │───▶│ 2.チャレンジ│───▶│ 3.認証器 │───▶│ 4.検証  │             │
│  │ 開始    │    │ 取得     │    │ で署名  │    │ & 保存  │             │
│  └─────────┘    └─────────┘    └─────────┘    └─────────┘             │
│       │              │              │              │                    │
│       ▼              ▼              ▼              ▼                    │
│  associateWebAuthn  Cognito から   ユーザーが     Cognito に           │
│  AccessToken()      チャレンジ取得  生体認証等で   公開鍵を登録         │
│                                    承認                                 │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### WebAuthn のサインインフロー

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     パスキーサインインフロー                             │
│                                                                          │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐             │
│  │ 1.開始  │───▶│ 2.チャレンジ│───▶│ 3.認証器 │───▶│ 4.検証  │             │
│  │         │    │ 取得     │    │ で署名  │    │ & 認証  │             │
│  └─────────┘    └─────────┘    └─────────┘    └─────────┘             │
│       │              │              │              │                    │
│       ▼              ▼              ▼              ▼                    │
│  signIn() with     Cognito から   ユーザーが     署名を検証して        │
│  preferredChallenge チャレンジ取得  生体認証等で   JWT トークン発行     │
│  = 'WEB_AUTHN'                     承認                                 │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

## 概念の説明

### Cognito の WebAuthn 設定

Cognito でパスキーを使用するには、以下の設定が必要です：

1. User Pool で WebAuthn を有効化
2. Relying Party ID の設定（通常はドメイン名）
3. User Verification の設定（required/preferred/discouraged）

```typescript
// CDK での WebAuthn 設定
const cfnUserPool = userPool.node.defaultChild as cognito.CfnUserPool;
cfnUserPool.addPropertyOverride("WebAuthnConfiguration", {
  RelyingPartyId: "taskflow.example.com",
  UserVerification: "preferred",
});
```

### Amplify Auth の WebAuthn API

```typescript
// パスキー登録
import { associateWebAuthnCredential } from "aws-amplify/auth";
await associateWebAuthnCredential();

// パスキーでサインイン
import { signIn } from "aws-amplify/auth";
const result = await signIn({
  username: email,
  options: {
    authFlowType: "USER_AUTH",
    preferredChallenge: "WEB_AUTHN",
  },
});

// パスキー一覧取得
import { listWebAuthnCredentials } from "aws-amplify/auth";
const credentials = await listWebAuthnCredentials();

// パスキー削除
import { deleteWebAuthnCredential } from "aws-amplify/auth";
await deleteWebAuthnCredential({ credentialId: "xxx" });
```

## コードサンプル

### CDK での WebAuthn 設定

```typescript
// infra/lib/cognito-stack.ts（WebAuthn 設定を追加）
import * as cdk from "aws-cdk-lib";
import * as cognito from "aws-cdk-lib/aws-cognito";
import { Construct } from "constructs";

interface CognitoStackProps extends cdk.StackProps {
  domainName: string; // 例: 'taskflow.example.com'
}

export class CognitoStack extends cdk.Stack {
  public readonly userPool: cognito.UserPool;
  public readonly userPoolClient: cognito.UserPoolClient;

  constructor(scope: Construct, id: string, props: CognitoStackProps) {
    super(scope, id, props);

    this.userPool = new cognito.UserPool(this, "TaskFlowUserPool", {
      userPoolName: "taskflow-user-pool",

      // ... 既存の設定 ...

      // サインインエイリアス
      signInAliases: {
        email: true,
      },
    });

    // WebAuthn 設定（L1 Construct を使用）
    const cfnUserPool = this.userPool.node.defaultChild as cognito.CfnUserPool;

    // WebAuthn Configuration
    cfnUserPool.addPropertyOverride("WebAuthnConfiguration", {
      RelyingPartyId: props.domainName,
      UserVerification: "preferred", // 'required' | 'preferred' | 'discouraged'
    });

    // App Client（WebAuthn 対応）
    this.userPoolClient = this.userPool.addClient("TaskFlowWebClient", {
      userPoolClientName: "taskflow-web-client",

      authFlows: {
        userPassword: true,
        userSrp: true,
        custom: true,
      },

      // 明示的な認証フローの設定
      authSessionValidity: cdk.Duration.minutes(3),

      generateSecret: false,
      preventUserExistenceErrors: true,
    });

    // App Client に USER_AUTH フローを追加（L1 で設定）
    const cfnUserPoolClient = this.userPoolClient.node
      .defaultChild as cognito.CfnUserPoolClient;
    cfnUserPoolClient.addPropertyOverride("ExplicitAuthFlows", [
      "ALLOW_USER_PASSWORD_AUTH",
      "ALLOW_USER_SRP_AUTH",
      "ALLOW_CUSTOM_AUTH",
      "ALLOW_USER_AUTH", // WebAuthn に必要
      "ALLOW_REFRESH_TOKEN_AUTH",
    ]);
  }
}
```

### AuthService への WebAuthn メソッド追加

```typescript
// apps/web/src/app/auth/services/auth.service.ts に追加
import {
  associateWebAuthnCredential,
  listWebAuthnCredentials,
  deleteWebAuthnCredential,
  signIn,
  confirmSignIn,
  type WebAuthnCredential,
} from 'aws-amplify/auth';

// AuthService クラスに追加

/**
 * パスキーを登録
 */
async registerPasskey(): Promise<void> {
  this._isLoading.set(true);
  this._error.set(null);

  try {
    await associateWebAuthnCredential();
  } catch (error) {
    const message = error instanceof Error ? error.message : 'パスキーの登録に失敗しました';
    this._error.set(message);
    throw error;
  } finally {
    this._isLoading.set(false);
  }
}

/**
 * 登録済みパスキー一覧を取得
 */
async listPasskeys(): Promise<WebAuthnCredential[]> {
  try {
    const result = await listWebAuthnCredentials();
    return result.credentials;
  } catch (error) {
    console.error('Failed to list passkeys:', error);
    return [];
  }
}

/**
 * パスキーを削除
 */
async deletePasskey(credentialId: string): Promise<void> {
  this._isLoading.set(true);
  this._error.set(null);

  try {
    await deleteWebAuthnCredential({ credentialId });
  } catch (error) {
    const message = error instanceof Error ? error.message : 'パスキーの削除に失敗しました';
    this._error.set(message);
    throw error;
  } finally {
    this._isLoading.set(false);
  }
}

/**
 * パスキーでサインイン
 */
async signInWithPasskey(email: string): Promise<void> {
  this._isLoading.set(true);
  this._error.set(null);

  try {
    const result = await signIn({
      username: email,
      options: {
        authFlowType: 'USER_AUTH',
        preferredChallenge: 'WEB_AUTHN',
      },
    });

    if (result.isSignedIn) {
      await this.checkAuthState();
    } else if (result.nextStep.signInStep === 'CONFIRM_SIGN_IN_WITH_WEB_AUTHN') {
      // WebAuthn チャレンジに応答
      const confirmResult = await confirmSignIn();
      if (confirmResult.isSignedIn) {
        await this.checkAuthState();
      }
    }
  } catch (error) {
    const message = error instanceof Error ? error.message : 'パスキーでのログインに失敗しました';
    this._error.set(message);
    throw error;
  } finally {
    this._isLoading.set(false);
  }
}
```

### パスキー登録コンポーネント

```typescript
// apps/web/src/app/auth/components/setup-passkey/setup-passkey.component.ts
import {
  Component,
  inject,
  signal,
  OnInit,
  ChangeDetectionStrategy,
} from "@angular/core";
import { CommonModule } from "@angular/common";
import { Router } from "@angular/router";
import { AuthService } from "../../services/auth.service";
import type { WebAuthnCredential } from "aws-amplify/auth";

@Component({
  selector: "app-setup-passkey",
  standalone: true,
  imports: [CommonModule],
  template: `
    <div class="setup-container">
      <h1>パスキーの設定</h1>

      @if (authService.error()) {
      <div class="error-message">{{ authService.error() }}</div>
      }

      <section class="info-section">
        <h2>パスキーとは？</h2>
        <p>
          パスキーは、パスワードの代わりに指紋認証や顔認証、
          セキュリティキーを使用してログインできる安全な認証方法です。
        </p>
        <ul>
          <li>✅ パスワードを覚える必要がない</li>
          <li>✅ フィッシング攻撃に強い</li>
          <li>✅ 高速でシームレスなログイン</li>
        </ul>
      </section>

      <section class="passkeys-section">
        <h2>登録済みのパスキー</h2>

        @if (loading()) {
        <p>読み込み中...</p>
        } @else if (passkeys().length === 0) {
        <p class="no-passkeys">パスキーが登録されていません。</p>
        } @else {
        <ul class="passkey-list">
          @for (passkey of passkeys(); track passkey.credentialId) {
          <li class="passkey-item">
            <div class="passkey-info">
              <span class="passkey-name">🔑 {{ getPasskeyName(passkey) }}</span>
              <span class="passkey-date">
                登録日: {{ formatDate(passkey.createdAt) }}
              </span>
            </div>
            <button
              class="btn-danger btn-small"
              (click)="removePasskey(passkey.credentialId)"
              [disabled]="authService.isLoading()"
            >
              削除
            </button>
          </li>
          }
        </ul>
        }
      </section>

      <section class="action-section">
        <button
          class="btn-primary"
          (click)="addPasskey()"
          [disabled]="authService.isLoading()"
        >
          @if (authService.isLoading()) { 処理中... } @else {
          新しいパスキーを追加 }
        </button>
      </section>

      <button class="btn-link" (click)="goBack()">← 設定に戻る</button>
    </div>
  `,
  styles: [
    `
      .setup-container {
        max-width: 600px;
        margin: 2rem auto;
        padding: 2rem;
      }
      .info-section {
        background: #f0f7ff;
        padding: 1.5rem;
        border-radius: 8px;
        margin-bottom: 2rem;
      }
      .info-section ul {
        list-style: none;
        padding: 0;
      }
      .info-section li {
        margin: 0.5rem 0;
      }
      .passkeys-section {
        margin-bottom: 2rem;
      }
      .no-passkeys {
        color: #666;
        font-style: italic;
      }
      .passkey-list {
        list-style: none;
        padding: 0;
      }
      .passkey-item {
        display: flex;
        justify-content: space-between;
        align-items: center;
        padding: 1rem;
        background: #fff;
        border: 1px solid #ddd;
        border-radius: 4px;
        margin-bottom: 0.5rem;
      }
      .passkey-info {
        display: flex;
        flex-direction: column;
      }
      .passkey-name {
        font-weight: bold;
      }
      .passkey-date {
        font-size: 0.875rem;
        color: #666;
      }
      .btn-small {
        padding: 0.25rem 0.75rem;
        font-size: 0.875rem;
      }
      .btn-link {
        background: none;
        border: none;
        color: #007bff;
        cursor: pointer;
        padding: 0;
        margin-top: 1rem;
      }
    `,
  ],
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class SetupPasskeyComponent implements OnInit {
  readonly authService = inject(AuthService);
  private readonly router = inject(Router);

  readonly loading = signal(true);
  readonly passkeys = signal<WebAuthnCredential[]>([]);

  async ngOnInit(): Promise<void> {
    await this.loadPasskeys();
  }

  async loadPasskeys(): Promise<void> {
    this.loading.set(true);
    try {
      const credentials = await this.authService.listPasskeys();
      this.passkeys.set(credentials);
    } finally {
      this.loading.set(false);
    }
  }

  async addPasskey(): Promise<void> {
    try {
      await this.authService.registerPasskey();
      await this.loadPasskeys();
    } catch {
      // エラーは AuthService で処理済み
    }
  }

  async removePasskey(credentialId: string): Promise<void> {
    if (!confirm("このパスキーを削除しますか？")) {
      return;
    }

    try {
      await this.authService.deletePasskey(credentialId);
      await this.loadPasskeys();
    } catch {
      // エラーは AuthService で処理済み
    }
  }

  getPasskeyName(passkey: WebAuthnCredential): string {
    // 認証器の種類に基づいて名前を生成
    return passkey.friendlyCredentialName || "パスキー";
  }

  formatDate(dateString: string): string {
    return new Date(dateString).toLocaleDateString("ja-JP");
  }

  goBack(): void {
    this.router.navigate(["/settings/security"]);
  }
}
```

### パスキーサインインコンポーネント

```typescript
// apps/web/src/app/auth/components/sign-in/sign-in.component.ts を更新
// パスキーサインインボタンを追加

template: `
  <div class="auth-container">
    <h1>ログイン</h1>

    <!-- パスキーでログイン -->
    <div class="passkey-section">
      <button
        class="btn-passkey"
        (click)="signInWithPasskey()"
        [disabled]="authService.isLoading()"
      >
        🔑 パスキーでログイン
      </button>
      <div class="divider">
        <span>または</span>
      </div>
    </div>

    <!-- 従来のログインフォーム -->
    <form [formGroup]="form" (ngSubmit)="onSubmit()">
      <!-- ... 既存のフォーム ... -->
    </form>
  </div>
`,

// コンポーネントクラスに追加
async signInWithPasskey(): Promise<void> {
  const email = this.form.get('email')?.value;

  if (!email) {
    // メールアドレスが入力されていない場合はダイアログで入力を求める
    const inputEmail = prompt('メールアドレスを入力してください');
    if (!inputEmail) return;

    try {
      await this.authService.signInWithPasskey(inputEmail);
      await this.router.navigate(['/dashboard']);
    } catch {
      // エラーは AuthService で処理済み
    }
    return;
  }

  try {
    await this.authService.signInWithPasskey(email);
    await this.router.navigate(['/dashboard']);
  } catch {
    // エラーは AuthService で処理済み
  }
}
```

## 実装タスク

### タスク 1: パスキー対応の確認

ブラウザが WebAuthn をサポートしているか確認する関数を実装してください。

<details>
<summary>回答</summary>

```typescript
// apps/web/src/app/auth/utils/webauthn-support.ts
export function isWebAuthnSupported(): boolean {
  return (
    typeof window !== 'undefined' &&
    typeof window.PublicKeyCredential !== 'undefined'
  );
}

export async function isPasskeyAvailable(): Promise<boolean> {
  if (!isWebAuthnSupported()) {
    return false;
  }

  try {
    // プラットフォーム認証器（Touch ID、Face ID など）が利用可能か確認
    const available = await PublicKeyCredential.isUserVerifyingPlatformAuthenticatorAvailable();
    return available;
  } catch {
    return false;
  }
}

// コンポーネントでの使用例
@Component({...})
export class SignInComponent implements OnInit {
  readonly passkeyAvailable = signal(false);

  async ngOnInit(): Promise<void> {
    this.passkeyAvailable.set(await isPasskeyAvailable());
  }
}
```

</details>

## よくある間違い

### ❌ Relying Party ID の設定ミス

```typescript
// 悪い例: 本番ドメインと異なる Relying Party ID
cfnUserPool.addPropertyOverride("WebAuthnConfiguration", {
  RelyingPartyId: "localhost", // 本番では動作しない
});
```

### ✅ 環境に応じた Relying Party ID

```typescript
// 良い例: 環境変数で切り替え
const relyingPartyId =
  process.env.NODE_ENV === "production" ? "taskflow.example.com" : "localhost";

cfnUserPool.addPropertyOverride("WebAuthnConfiguration", {
  RelyingPartyId: relyingPartyId,
});
```

## まとめ

この章では以下を学びました：

- WebAuthn/FIDO2 の基本概念
- Cognito でのパスキー設定
- パスキーの登録・削除・一覧取得
- パスキーによるサインイン実装

## 確認クイズ

<details>
<summary>Q1: パスキーがパスワードより安全な理由は？</summary>

**A1:**

1. フィッシング耐性: パスキーは特定のドメインに紐づくため、偽サイトでは使用できない
2. 秘密鍵の保護: 秘密鍵はデバイスから外に出ない
3. 推測不可能: ランダムな暗号鍵のため、ブルートフォース攻撃が不可能
4. 再利用不可: サイトごとに異なる鍵ペアを使用
</details>

<details>
<summary>Q2: Relying Party ID とは何ですか？</summary>

**A2:**
Relying Party ID は、パスキーが紐づくドメインを識別する文字列です。通常はアプリケーションのドメイン名（例: `taskflow.example.com`）を使用します。

パスキーは Relying Party ID に紐づくため、異なるドメインでは使用できません。これがフィッシング耐性の基盤となっています。

</details>

---

前の章: [05 - MFA（多要素認証）の実装](./05-mfa-implementation.md)
次の章: [07 - ソーシャルログインの統合](./07-social-login.md)
