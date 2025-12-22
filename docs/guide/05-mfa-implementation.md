# 05 - MFA（多要素認証）の実装

## 学習の目的

- MFA の種類（TOTP、SMS、Email）と特徴を理解する
- Cognito での MFA 設定方法を学ぶ
- TOTP（Google Authenticator など）の設定フローを実装する
- MFA チャレンジへの対応を実装する

## 背景知識

### MFA とは

MFA（Multi-Factor Authentication）は、複数の認証要素を組み合わせてセキュリティを強化する仕組みです：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        認証の 3 要素                                     │
│                                                                          │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────┐ │
│  │ 知識要素        │  │ 所持要素        │  │ 生体要素                │ │
│  │ (Something      │  │ (Something      │  │ (Something you are)     │ │
│  │  you know)      │  │  you have)      │  │                         │ │
│  │                 │  │                 │  │                         │ │
│  │ • パスワード    │  │ • スマートフォン│  │ • 指紋                  │ │
│  │ • PIN           │  │ • セキュリティ  │  │ • 顔認証                │ │
│  │ • 秘密の質問    │  │   キー          │  │ • 虹彩                  │ │
│  │                 │  │ • ハードウェア  │  │                         │ │
│  │                 │  │   トークン      │  │                         │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────────────┘ │
│                                                                          │
│  MFA = 上記のうち 2 つ以上を組み合わせる                                 │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Cognito がサポートする MFA

| 種類  | 説明                             | 特徴                                        |
| ----- | -------------------------------- | ------------------------------------------- |
| TOTP  | 時間ベースのワンタイムパスワード | Google Authenticator 等で生成、オフライン可 |
| SMS   | SMS でコードを送信               | 電話番号が必要、コストがかかる              |
| Email | メールでコードを送信             | メールアドレスが必要                        |

### MFA 設定モード

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        MFA 設定モード                                    │
│                                                                          │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────┐ │
│  │ OFF             │  │ OPTIONAL        │  │ ON                      │ │
│  │                 │  │                 │  │                         │ │
│  │ MFA を使用しない│  │ ユーザーが選択  │  │ 全ユーザー必須          │ │
│  │                 │  │ 可能            │  │                         │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────────────┘ │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

## 概念の説明

### TOTP の仕組み

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        TOTP 設定フロー                                   │
│                                                                          │
│  1. シークレットキー生成                                                 │
│     ┌─────────┐    ┌─────────┐                                         │
│     │ Cognito │───▶│ Secret  │  Base32 エンコードされた秘密鍵          │
│     └─────────┘    │ Key     │                                         │
│                    └────┬────┘                                         │
│                         │                                               │
│  2. QR コード表示       ▼                                               │
│     ┌─────────────────────────────────────────┐                        │
│     │  otpauth://totp/TaskFlow:user@example   │                        │
│     │  ?secret=JBSWY3DPEHPK3PXP               │                        │
│     │  &issuer=TaskFlow                       │                        │
│     └─────────────────────────────────────────┘                        │
│                         │                                               │
│  3. Authenticator で読み取り                                            │
│     ┌─────────┐    ┌─────────┐                                         │
│     │ QR Code │───▶│ Google  │  アプリに登録                           │
│     └─────────┘    │ Auth    │                                         │
│                    └────┬────┘                                         │
│                         │                                               │
│  4. 検証コード入力      ▼                                               │
│     ┌─────────┐    ┌─────────┐    ┌─────────┐                         │
│     │ 6桁    │───▶│ Cognito │───▶│ 設定    │                         │
│     │ コード  │    │ 検証    │    │ 完了    │                         │
│     └─────────┘    └─────────┘    └─────────┘                         │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### サインイン時の MFA フロー

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     MFA サインインフロー                                 │
│                                                                          │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐             │
│  │ SignIn  │───▶│ MFA     │───▶│ コード  │───▶│ 認証    │             │
│  │ (email/ │    │ Challenge│    │ 入力    │    │ 完了    │             │
│  │ password)│    │         │    │         │    │         │             │
│  └─────────┘    └─────────┘    └─────────┘    └─────────┘             │
│       │              │              │              │                    │
│       ▼              ▼              ▼              ▼                    │
│  signIn()      nextStep:       confirmSignIn()  isSignedIn:           │
│               'CONFIRM_SIGN_IN                   true                  │
│               _WITH_TOTP_CODE'                                         │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

## コードサンプル

### CDK での MFA 設定

```typescript
// infra/lib/cognito-stack.ts（MFA 設定を追加）
import * as cdk from "aws-cdk-lib";
import * as cognito from "aws-cdk-lib/aws-cognito";
import { Construct } from "constructs";

export class CognitoStack extends cdk.Stack {
  public readonly userPool: cognito.UserPool;

  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    this.userPool = new cognito.UserPool(this, "TaskFlowUserPool", {
      userPoolName: "taskflow-user-pool",

      // ... 既存の設定 ...

      // MFA 設定
      mfa: cognito.Mfa.OPTIONAL, // ユーザーが選択可能
      mfaSecondFactor: {
        sms: false, // SMS MFA を無効化（コスト削減）
        otp: true, // TOTP を有効化
      },

      // Advanced Security（適応型認証）
      advancedSecurityMode: cognito.AdvancedSecurityMode.ENFORCED,
    });
  }
}
```

### AuthService への MFA メソッド追加

```typescript
// apps/web/src/app/auth/services/auth.service.ts に追加
import {
  setUpTOTP,
  verifyTOTPSetup,
  confirmSignIn,
  updateMFAPreference,
  fetchMFAPreference,
  type TOTPSetupDetails,
} from 'aws-amplify/auth';

// AuthService クラスに追加
/**
 * TOTP セットアップを開始
 */
async setupTOTP(): Promise<TOTPSetupDetails> {
  this._isLoading.set(true);
  this._error.set(null);

  try {
    const totpSetupDetails = await setUpTOTP();
    return totpSetupDetails;
  } catch (error) {
    const message = error instanceof Error ? error.message : 'TOTP セットアップに失敗しました';
    this._error.set(message);
    throw error;
  } finally {
    this._isLoading.set(false);
  }
}

/**
 * TOTP セットアップを検証
 */
async verifyTOTPSetup(code: string): Promise<void> {
  this._isLoading.set(true);
  this._error.set(null);

  try {
    await verifyTOTPSetup({ code });
    // TOTP を優先 MFA として設定
    await updateMFAPreference({ totp: 'PREFERRED' });
  } catch (error) {
    const message = error instanceof Error ? error.message : 'TOTP 検証に失敗しました';
    this._error.set(message);
    throw error;
  } finally {
    this._isLoading.set(false);
  }
}

/**
 * MFA チャレンジに応答（TOTP）
 */
async confirmSignInWithTOTP(code: string): Promise<void> {
  this._isLoading.set(true);
  this._error.set(null);

  try {
    const result = await confirmSignIn({ challengeResponse: code });
    if (result.isSignedIn) {
      await this.checkAuthState();
    }
  } catch (error) {
    const message = error instanceof Error ? error.message : 'MFA 認証に失敗しました';
    this._error.set(message);
    throw error;
  } finally {
    this._isLoading.set(false);
  }
}

/**
 * 現在の MFA 設定を取得
 */
async getMFAPreference(): Promise<{ enabled: string[]; preferred?: string }> {
  try {
    const preference = await fetchMFAPreference();
    return {
      enabled: preference.enabled || [],
      preferred: preference.preferred,
    };
  } catch {
    return { enabled: [] };
  }
}

/**
 * MFA を無効化
 */
async disableMFA(): Promise<void> {
  this._isLoading.set(true);
  this._error.set(null);

  try {
    await updateMFAPreference({ totp: 'DISABLED' });
  } catch (error) {
    const message = error instanceof Error ? error.message : 'MFA 無効化に失敗しました';
    this._error.set(message);
    throw error;
  } finally {
    this._isLoading.set(false);
  }
}
```

### TOTP セットアップコンポーネント

```typescript
// apps/web/src/app/auth/components/setup-totp/setup-totp.component.ts
import {
  Component,
  inject,
  signal,
  OnInit,
  ChangeDetectionStrategy,
} from "@angular/core";
import { CommonModule } from "@angular/common";
import { ReactiveFormsModule, FormBuilder, Validators } from "@angular/forms";
import { Router } from "@angular/router";
import { AuthService } from "../../services/auth.service";

@Component({
  selector: "app-setup-totp",
  standalone: true,
  imports: [CommonModule, ReactiveFormsModule],
  template: `
    <div class="setup-container">
      <h1>二要素認証の設定</h1>

      @if (authService.error()) {
      <div class="error-message">{{ authService.error() }}</div>
      } @if (!setupComplete()) { @if (qrCodeUri()) {
      <div class="setup-steps">
        <h2>ステップ 1: QR コードをスキャン</h2>
        <p>
          Google Authenticator などの認証アプリで以下の QR
          コードをスキャンしてください。
        </p>

        <div class="qr-code">
          <img [src]="qrCodeDataUrl()" alt="TOTP QR Code" />
        </div>

        <details class="manual-entry">
          <summary>手動で入力する場合</summary>
          <p>
            シークレットキー: <code>{{ secretKey() }}</code>
          </p>
        </details>

        <h2>ステップ 2: 確認コードを入力</h2>
        <p>認証アプリに表示されている 6 桁のコードを入力してください。</p>

        <form [formGroup]="form" (ngSubmit)="onVerify()">
          <div class="form-group">
            <input
              type="text"
              formControlName="code"
              placeholder="000000"
              maxlength="6"
              inputmode="numeric"
              autocomplete="one-time-code"
            />
          </div>

          <button
            type="submit"
            [disabled]="form.invalid || authService.isLoading()"
          >
            @if (authService.isLoading()) { 検証中... } @else { 設定を完了 }
          </button>
        </form>
      </div>
      } @else {
      <div class="loading">
        <p>セットアップを準備中...</p>
      </div>
      } } @else {
      <div class="success">
        <h2>✅ 設定完了</h2>
        <p>二要素認証が有効になりました。</p>
        <p>
          次回のログインから、パスワードに加えて認証アプリのコードが必要になります。
        </p>
        <button (click)="goToDashboard()">ダッシュボードへ</button>
      </div>
      }
    </div>
  `,
  styles: [
    `
      .setup-container {
        max-width: 500px;
        margin: 2rem auto;
        padding: 2rem;
      }
      .qr-code {
        display: flex;
        justify-content: center;
        margin: 1.5rem 0;
      }
      .qr-code img {
        width: 200px;
        height: 200px;
      }
      .manual-entry {
        margin: 1rem 0;
        padding: 1rem;
        background: #f5f5f5;
        border-radius: 4px;
      }
      .manual-entry code {
        font-family: monospace;
        background: #fff;
        padding: 0.25rem 0.5rem;
        border-radius: 2px;
      }
      input[type="text"] {
        font-size: 1.5rem;
        text-align: center;
        letter-spacing: 0.5rem;
        padding: 1rem;
      }
      .success {
        text-align: center;
      }
    `,
  ],
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class SetupTotpComponent implements OnInit {
  readonly authService = inject(AuthService);
  private readonly fb = inject(FormBuilder);
  private readonly router = inject(Router);

  readonly qrCodeUri = signal("");
  readonly qrCodeDataUrl = signal("");
  readonly secretKey = signal("");
  readonly setupComplete = signal(false);

  readonly form = this.fb.nonNullable.group({
    code: [
      "",
      [Validators.required, Validators.minLength(6), Validators.maxLength(6)],
    ],
  });

  async ngOnInit(): Promise<void> {
    try {
      const setupDetails = await this.authService.setupTOTP();

      // QR コード URI を生成
      const user = this.authService.user();
      const uri = setupDetails.getSetupUri("TaskFlow", user?.email || "user");
      this.qrCodeUri.set(uri.toString());
      this.secretKey.set(setupDetails.sharedSecret);

      // QR コードを生成（qrcode ライブラリを使用）
      const QRCode = await import("qrcode");
      const dataUrl = await QRCode.toDataURL(uri.toString());
      this.qrCodeDataUrl.set(dataUrl);
    } catch {
      // エラーは AuthService で処理済み
    }
  }

  async onVerify(): Promise<void> {
    if (this.form.invalid) return;

    const { code } = this.form.getRawValue();

    try {
      await this.authService.verifyTOTPSetup(code);
      this.setupComplete.set(true);
    } catch {
      // エラーは AuthService で処理済み
    }
  }

  goToDashboard(): void {
    this.router.navigate(["/dashboard"]);
  }
}
```

### MFA チャレンジコンポーネント

```typescript
// apps/web/src/app/auth/components/mfa-totp/mfa-totp.component.ts
import { Component, inject, ChangeDetectionStrategy } from "@angular/core";
import { CommonModule } from "@angular/common";
import { ReactiveFormsModule, FormBuilder, Validators } from "@angular/forms";
import { Router } from "@angular/router";
import { AuthService } from "../../services/auth.service";

@Component({
  selector: "app-mfa-totp",
  standalone: true,
  imports: [CommonModule, ReactiveFormsModule],
  template: `
    <div class="auth-container">
      <h1>二要素認証</h1>
      <p>認証アプリに表示されている 6 桁のコードを入力してください。</p>

      @if (authService.error()) {
      <div class="error-message">{{ authService.error() }}</div>
      }

      <form [formGroup]="form" (ngSubmit)="onSubmit()">
        <div class="form-group">
          <input
            type="text"
            formControlName="code"
            placeholder="000000"
            maxlength="6"
            inputmode="numeric"
            autocomplete="one-time-code"
            autofocus
          />
        </div>

        <button
          type="submit"
          [disabled]="form.invalid || authService.isLoading()"
        >
          @if (authService.isLoading()) { 確認中... } @else { 確認 }
        </button>
      </form>
    </div>
  `,
  styles: [
    `
      input[type="text"] {
        font-size: 1.5rem;
        text-align: center;
        letter-spacing: 0.5rem;
        padding: 1rem;
      }
    `,
  ],
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class MfaTotpComponent {
  readonly authService = inject(AuthService);
  private readonly fb = inject(FormBuilder);
  private readonly router = inject(Router);

  readonly form = this.fb.nonNullable.group({
    code: [
      "",
      [Validators.required, Validators.minLength(6), Validators.maxLength(6)],
    ],
  });

  async onSubmit(): Promise<void> {
    if (this.form.invalid) return;

    const { code } = this.form.getRawValue();

    try {
      await this.authService.confirmSignInWithTOTP(code);
      await this.router.navigate(["/dashboard"]);
    } catch {
      // エラーは AuthService で処理済み
      this.form.reset();
    }
  }
}
```

## 実装タスク

### タスク 1: MFA 設定管理画面の実装

ユーザーが MFA の有効/無効を切り替えられる設定画面を実装してください。

<details>
<summary>回答</summary>

```typescript
// apps/web/src/app/settings/security/security-settings.component.ts
import {
  Component,
  inject,
  signal,
  OnInit,
  ChangeDetectionStrategy,
} from "@angular/core";
import { CommonModule } from "@angular/common";
import { Router } from "@angular/router";
import { AuthService } from "../../auth/services/auth.service";

@Component({
  selector: "app-security-settings",
  standalone: true,
  imports: [CommonModule],
  template: `
    <div class="settings-container">
      <h1>セキュリティ設定</h1>

      <section class="setting-section">
        <h2>二要素認証（MFA）</h2>

        @if (loading()) {
        <p>読み込み中...</p>
        } @else {
        <div class="mfa-status">
          <span class="status-badge" [class.enabled]="mfaEnabled()">
            {{ mfaEnabled() ? "有効" : "無効" }}
          </span>
        </div>

        @if (mfaEnabled()) {
        <p>認証アプリによる二要素認証が有効です。</p>
        <button
          class="btn-danger"
          (click)="disableMFA()"
          [disabled]="authService.isLoading()"
        >
          MFA を無効にする
        </button>
        } @else {
        <p>二要素認証を有効にすると、アカウントのセキュリティが向上します。</p>
        <button class="btn-primary" (click)="setupMFA()">MFA を設定する</button>
        } }
      </section>

      <section class="setting-section">
        <h2>パスキー</h2>
        <p>パスキーを使用すると、パスワードなしでログインできます。</p>
        <button class="btn-primary" (click)="setupPasskey()">
          パスキーを追加
        </button>
      </section>
    </div>
  `,
  styles: [
    `
      .settings-container {
        max-width: 600px;
        margin: 2rem auto;
        padding: 2rem;
      }
      .setting-section {
        margin-bottom: 2rem;
        padding: 1.5rem;
        background: #fff;
        border-radius: 8px;
        box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
      }
      .status-badge {
        display: inline-block;
        padding: 0.25rem 0.75rem;
        border-radius: 4px;
        background: #f0f0f0;
        color: #666;
      }
      .status-badge.enabled {
        background: #d4edda;
        color: #155724;
      }
      .btn-danger {
        background: #dc3545;
        color: white;
      }
    `,
  ],
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class SecuritySettingsComponent implements OnInit {
  readonly authService = inject(AuthService);
  private readonly router = inject(Router);

  readonly loading = signal(true);
  readonly mfaEnabled = signal(false);

  async ngOnInit(): Promise<void> {
    await this.loadMFAStatus();
  }

  async loadMFAStatus(): Promise<void> {
    this.loading.set(true);
    try {
      const preference = await this.authService.getMFAPreference();
      this.mfaEnabled.set(preference.enabled.includes("TOTP"));
    } finally {
      this.loading.set(false);
    }
  }

  setupMFA(): void {
    this.router.navigate(["/settings/setup-totp"]);
  }

  async disableMFA(): Promise<void> {
    if (
      !confirm("MFA を無効にしますか？アカウントのセキュリティが低下します。")
    ) {
      return;
    }

    try {
      await this.authService.disableMFA();
      this.mfaEnabled.set(false);
    } catch {
      // エラーは AuthService で処理済み
    }
  }

  setupPasskey(): void {
    this.router.navigate(["/settings/setup-passkey"]);
  }
}
```

</details>

## よくある間違い

### ❌ シークレットキーをログに出力

```typescript
// 悪い例: シークレットキーをログに出力
console.log("TOTP Secret:", setupDetails.sharedSecret);
```

### ✅ シークレットキーは安全に扱う

```typescript
// 良い例: シークレットキーは表示のみ、ログには出力しない
this.secretKey.set(setupDetails.sharedSecret);
```

## まとめ

この章では以下を学びました：

- MFA の種類と Cognito での設定方法
- TOTP セットアップフローの実装
- MFA チャレンジへの対応
- MFA 設定管理画面の実装

## 確認クイズ

<details>
<summary>Q1: TOTP と SMS MFA の違いは何ですか？</summary>

**A1:**

- TOTP: 認証アプリで生成、オフラインでも使用可能、コストがかからない
- SMS: 電話番号に送信、ネットワーク接続が必要、SMS 送信コストがかかる

セキュリティ面では、SMS は SIM スワップ攻撃のリスクがあるため、TOTP の方が推奨されます。

</details>

<details>
<summary>Q2: MFA 設定モードの OPTIONAL と ON の違いは？</summary>

**A2:**

- OPTIONAL: ユーザーが MFA を有効にするかどうかを選択できる
- ON: 全ユーザーに MFA を強制する

OPTIONAL は段階的な導入に適しており、ON は高セキュリティが必要な場合に使用します。

</details>

---

前の章: [04 - Hono API と JWT 検証](./04-hono-api-jwt.md)
次の章: [06 - パスキー認証の実装](./06-passkey-authentication.md)
