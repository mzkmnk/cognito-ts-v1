# 02 - 基本認証フローの実装

## 学習の目的

- AWS Amplify Auth ライブラリの使い方を理解する
- サインアップ、サインイン、サインアウトのフローを実装する
- メール検証フローを実装する
- トークンの取得と管理方法を学ぶ

## 背景知識

### Amplify Auth vs AWS SDK

Cognito との通信には主に 2 つの選択肢があります：

| 方法                                | 特徴                       | ユースケース               |
| ----------------------------------- | -------------------------- | -------------------------- |
| AWS Amplify Auth                    | 高レベル API、状態管理込み | フロントエンド、モバイル   |
| AWS SDK (cognito-identity-provider) | 低レベル API、細かい制御   | バックエンド、カスタム実装 |

このガイドでは、フロントエンドに Amplify Auth、バックエンドに AWS SDK を使用します。

### 認証フローの流れ

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      サインアップフロー                                  │
│                                                                          │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐             │
│  │ 入力    │───▶│ SignUp  │───▶│ 検証    │───▶│ 完了    │             │
│  │ フォーム│    │ API     │    │ コード  │    │         │             │
│  └─────────┘    └─────────┘    └─────────┘    └─────────┘             │
│       │              │              │              │                    │
│       ▼              ▼              ▼              ▼                    │
│  email,         Cognito に     メールで      confirmSignUp            │
│  password       ユーザー作成   コード送信    API で検証                │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

┌─────────────────────────────────────────────────────────────────────────┐
│ サインインフロー │
│ │
│ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ │
│ │ 入力 │───▶│ SignIn │───▶│ MFA │───▶│ 完了 │ │
│ │ フォーム │ │ API │ │ (任意) │ │ │ │
│ └─────────┘ └─────────┘ └─────────┘ └─────────┘ │
│ │ │ │ │ │
│ ▼ ▼ ▼ ▼ │
│ email, 認証情報を MFA が有効なら JWT トークン │
│ password 検証 チャレンジ を取得 │
│ │
└─────────────────────────────────────────────────────────────────────────┘

````

## 概念の説明

### Amplify Auth の設定

Amplify Auth は `Amplify.configure()` で初期化します：

```typescript
import { Amplify } from 'aws-amplify';

Amplify.configure({
  Auth: {
    Cognito: {
      userPoolId: 'ap-northeast-1_xxxxxxxx',
      userPoolClientId: 'xxxxxxxxxxxxxxxxxxxxxxxxxx',
      signUpVerificationMethod: 'code',
    }
  }
});
````

### 認証状態の管理

Amplify Auth は内部でトークンを管理し、自動的にリフレッシュします：

```typescript
import { getCurrentUser, fetchAuthSession } from "aws-amplify/auth";

// 現在のユーザー情報を取得
const user = await getCurrentUser();

// セッション（トークン）を取得
const session = await fetchAuthSession();
const idToken = session.tokens?.idToken;
const accessToken = session.tokens?.accessToken;
```

## コードサンプル

### Amplify のインストールと設定

```bash
# apps/web ディレクトリで
cd apps/web
npm install aws-amplify
```

### 認証サービスの実装

```typescript
// apps/web/src/app/auth/services/auth.service.ts
import { Injectable, signal, computed } from "@angular/core";
import {
  signUp,
  confirmSignUp,
  signIn,
  signOut,
  getCurrentUser,
  fetchAuthSession,
  resendSignUpCode,
  type SignUpOutput,
  type SignInOutput,
} from "aws-amplify/auth";

export interface AuthUser {
  userId: string;
  email: string;
  emailVerified: boolean;
}

export interface AuthState {
  user: AuthUser | null;
  isAuthenticated: boolean;
  isLoading: boolean;
  error: string | null;
}

@Injectable({
  providedIn: "root",
})
export class AuthService {
  // Signals for reactive state management
  private readonly _user = signal<AuthUser | null>(null);
  private readonly _isLoading = signal(true);
  private readonly _error = signal<string | null>(null);

  // Computed values
  readonly user = this._user.asReadonly();
  readonly isLoading = this._isLoading.asReadonly();
  readonly error = this._error.asReadonly();
  readonly isAuthenticated = computed(() => this._user() !== null);

  constructor() {
    this.checkAuthState();
  }

  /**
   * 現在の認証状態を確認
   */
  async checkAuthState(): Promise<void> {
    this._isLoading.set(true);
    this._error.set(null);

    try {
      const currentUser = await getCurrentUser();
      const session = await fetchAuthSession();

      const idToken = session.tokens?.idToken;
      const claims = idToken?.payload;

      this._user.set({
        userId: currentUser.userId,
        email: claims?.email as string,
        emailVerified: claims?.email_verified as boolean,
      });
    } catch {
      // ユーザーが認証されていない場合
      this._user.set(null);
    } finally {
      this._isLoading.set(false);
    }
  }

  /**
   * サインアップ
   */
  async signUp(email: string, password: string): Promise<SignUpOutput> {
    this._isLoading.set(true);
    this._error.set(null);

    try {
      const result = await signUp({
        username: email,
        password,
        options: {
          userAttributes: {
            email,
          },
        },
      });
      return result;
    } catch (error) {
      const message =
        error instanceof Error ? error.message : "登録に失敗しました";
      this._error.set(message);
      throw error;
    } finally {
      this._isLoading.set(false);
    }
  }

  /**
   * サインアップ確認（メール検証）
   */
  async confirmSignUp(email: string, code: string): Promise<void> {
    this._isLoading.set(true);
    this._error.set(null);

    try {
      await confirmSignUp({
        username: email,
        confirmationCode: code,
      });
    } catch (error) {
      const message =
        error instanceof Error ? error.message : "確認に失敗しました";
      this._error.set(message);
      throw error;
    } finally {
      this._isLoading.set(false);
    }
  }

  /**
   * 確認コードの再送信
   */
  async resendConfirmationCode(email: string): Promise<void> {
    this._isLoading.set(true);
    this._error.set(null);

    try {
      await resendSignUpCode({ username: email });
    } catch (error) {
      const message =
        error instanceof Error ? error.message : "コードの再送信に失敗しました";
      this._error.set(message);
      throw error;
    } finally {
      this._isLoading.set(false);
    }
  }

  /**
   * サインイン
   */
  async signIn(email: string, password: string): Promise<SignInOutput> {
    this._isLoading.set(true);
    this._error.set(null);

    try {
      const result = await signIn({
        username: email,
        password,
      });

      if (result.isSignedIn) {
        await this.checkAuthState();
      }

      return result;
    } catch (error) {
      const message =
        error instanceof Error ? error.message : "ログインに失敗しました";
      this._error.set(message);
      throw error;
    } finally {
      this._isLoading.set(false);
    }
  }

  /**
   * サインアウト
   */
  async signOut(): Promise<void> {
    this._isLoading.set(true);
    this._error.set(null);

    try {
      await signOut();
      this._user.set(null);
    } catch (error) {
      const message =
        error instanceof Error ? error.message : "ログアウトに失敗しました";
      this._error.set(message);
      throw error;
    } finally {
      this._isLoading.set(false);
    }
  }

  /**
   * アクセストークンを取得
   */
  async getAccessToken(): Promise<string | undefined> {
    try {
      const session = await fetchAuthSession();
      return session.tokens?.accessToken?.toString();
    } catch {
      return undefined;
    }
  }

  /**
   * ID トークンを取得
   */
  async getIdToken(): Promise<string | undefined> {
    try {
      const session = await fetchAuthSession();
      return session.tokens?.idToken?.toString();
    } catch {
      return undefined;
    }
  }
}
```

### Amplify 設定ファイル

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
    userPoolId: "ap-northeast-1_xxxxxxxx", // CDK 出力から取得
    userPoolClientId: "xxxxxxxxxxxxxxxxxxxxxxxxxx", // CDK 出力から取得
  },
  apiUrl: "http://localhost:3000",
};
```

### アプリケーション初期化

```typescript
// apps/web/src/app/app.config.ts
import { ApplicationConfig, APP_INITIALIZER } from "@angular/core";
import { provideRouter } from "@angular/router";
import { provideHttpClient, withInterceptors } from "@angular/common/http";
import { routes } from "./app.routes";
import { configureAmplify } from "./auth/amplify-config";
import { authInterceptor } from "./auth/interceptors/auth.interceptor";

// Amplify を初期化する関数
function initializeApp(): () => Promise<void> {
  return () => {
    configureAmplify();
    return Promise.resolve();
  };
}

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    provideHttpClient(withInterceptors([authInterceptor])),
    {
      provide: APP_INITIALIZER,
      useFactory: initializeApp,
      multi: true,
    },
  ],
};
```

## 実装タスク

### タスク 1: サインアップコンポーネントの実装

以下の要件を満たすサインアップコンポーネントを実装してください：

**要件:**

- メールアドレスとパスワードの入力フォーム
- パスワード確認フィールド
- バリデーション（メール形式、パスワード強度）
- エラーメッセージの表示
- サインアップ成功後、確認コード入力画面へ遷移

<details>
<summary>ヒント</summary>

Angular の Reactive Forms を使用し、`AuthService.signUp()` を呼び出します。
`signUp` の結果で `nextStep.signUpStep` が `'CONFIRM_SIGN_UP'` の場合、確認コード入力が必要です。

</details>

<details>
<summary>回答</summary>

```typescript
// apps/web/src/app/auth/components/sign-up/sign-up.component.ts
import { Component, inject, signal } from "@angular/core";
import { CommonModule } from "@angular/common";
import { ReactiveFormsModule, FormBuilder, Validators } from "@angular/forms";
import { Router, RouterLink } from "@angular/router";
import { AuthService } from "../../services/auth.service";

@Component({
  selector: "app-sign-up",
  standalone: true,
  imports: [CommonModule, ReactiveFormsModule, RouterLink],
  template: `
    <div class="auth-container">
      <h1>アカウント作成</h1>

      @if (authService.error()) {
      <div class="error-message">{{ authService.error() }}</div>
      }

      <form [formGroup]="form" (ngSubmit)="onSubmit()">
        <div class="form-group">
          <label for="email">メールアドレス</label>
          <input
            id="email"
            type="email"
            formControlName="email"
            [class.invalid]="
              form.controls.email.invalid && form.controls.email.touched
            "
          />
          @if (form.controls.email.errors?.['required'] &&
          form.controls.email.touched) {
          <span class="field-error">メールアドレスは必須です</span>
          } @if (form.controls.email.errors?.['email'] &&
          form.controls.email.touched) {
          <span class="field-error"
            >有効なメールアドレスを入力してください</span
          >
          }
        </div>

        <div class="form-group">
          <label for="password">パスワード</label>
          <input
            id="password"
            type="password"
            formControlName="password"
            [class.invalid]="
              form.controls.password.invalid && form.controls.password.touched
            "
          />
          @if (form.controls.password.errors?.['required'] &&
          form.controls.password.touched) {
          <span class="field-error">パスワードは必須です</span>
          } @if (form.controls.password.errors?.['minlength'] &&
          form.controls.password.touched) {
          <span class="field-error">パスワードは8文字以上必要です</span>
          }
        </div>

        <div class="form-group">
          <label for="confirmPassword">パスワード（確認）</label>
          <input
            id="confirmPassword"
            type="password"
            formControlName="confirmPassword"
            [class.invalid]="
              form.controls.confirmPassword.invalid &&
              form.controls.confirmPassword.touched
            "
          />
          @if (passwordMismatch()) {
          <span class="field-error">パスワードが一致しません</span>
          }
        </div>

        <button
          type="submit"
          [disabled]="
            form.invalid || authService.isLoading() || passwordMismatch()
          "
        >
          @if (authService.isLoading()) {
          <span>処理中...</span>
          } @else {
          <span>登録</span>
          }
        </button>
      </form>

      <p class="auth-link">
        すでにアカウントをお持ちですか？
        <a routerLink="/auth/sign-in">ログイン</a>
      </p>
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class SignUpComponent {
  readonly authService = inject(AuthService);
  private readonly fb = inject(FormBuilder);
  private readonly router = inject(Router);

  readonly form = this.fb.nonNullable.group({
    email: ["", [Validators.required, Validators.email]],
    password: ["", [Validators.required, Validators.minLength(8)]],
    confirmPassword: ["", [Validators.required]],
  });

  readonly passwordMismatch = signal(false);

  constructor() {
    // パスワード一致チェック
    this.form.valueChanges.subscribe(() => {
      const { password, confirmPassword } = this.form.value;
      this.passwordMismatch.set(
        password !== confirmPassword && !!confirmPassword
      );
    });
  }

  async onSubmit(): Promise<void> {
    if (this.form.invalid || this.passwordMismatch()) return;

    const { email, password } = this.form.getRawValue();

    try {
      const result = await this.authService.signUp(email, password);

      if (result.nextStep.signUpStep === "CONFIRM_SIGN_UP") {
        // 確認コード入力画面へ遷移
        await this.router.navigate(["/auth/confirm-sign-up"], {
          queryParams: { email },
        });
      }
    } catch {
      // エラーは AuthService で処理済み
    }
  }
}
```

</details>

### タスク 2: 確認コード入力コンポーネントの実装

サインアップ後のメール検証コード入力コンポーネントを実装してください。

<details>
<summary>回答</summary>

```typescript
// apps/web/src/app/auth/components/confirm-sign-up/confirm-sign-up.component.ts
import {
  Component,
  inject,
  OnInit,
  signal,
  ChangeDetectionStrategy,
} from "@angular/core";
import { CommonModule } from "@angular/common";
import { ReactiveFormsModule, FormBuilder, Validators } from "@angular/forms";
import { Router, ActivatedRoute, RouterLink } from "@angular/router";
import { AuthService } from "../../services/auth.service";

@Component({
  selector: "app-confirm-sign-up",
  standalone: true,
  imports: [CommonModule, ReactiveFormsModule, RouterLink],
  template: `
    <div class="auth-container">
      <h1>メール確認</h1>
      <p class="description">
        {{ email() }} に送信された確認コードを入力してください
      </p>

      @if (authService.error()) {
      <div class="error-message">{{ authService.error() }}</div>
      } @if (resendSuccess()) {
      <div class="success-message">確認コードを再送信しました</div>
      }

      <form [formGroup]="form" (ngSubmit)="onSubmit()">
        <div class="form-group">
          <label for="code">確認コード</label>
          <input
            id="code"
            type="text"
            formControlName="code"
            placeholder="123456"
            maxlength="6"
          />
        </div>

        <button
          type="submit"
          [disabled]="form.invalid || authService.isLoading()"
        >
          @if (authService.isLoading()) {
          <span>確認中...</span>
          } @else {
          <span>確認</span>
          }
        </button>
      </form>

      <button
        type="button"
        class="link-button"
        (click)="resendCode()"
        [disabled]="authService.isLoading()"
      >
        確認コードを再送信
      </button>

      <p class="auth-link">
        <a routerLink="/auth/sign-up">登録画面に戻る</a>
      </p>
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class ConfirmSignUpComponent implements OnInit {
  readonly authService = inject(AuthService);
  private readonly fb = inject(FormBuilder);
  private readonly router = inject(Router);
  private readonly route = inject(ActivatedRoute);

  readonly email = signal("");
  readonly resendSuccess = signal(false);

  readonly form = this.fb.nonNullable.group({
    code: [
      "",
      [Validators.required, Validators.minLength(6), Validators.maxLength(6)],
    ],
  });

  ngOnInit(): void {
    const email = this.route.snapshot.queryParamMap.get("email");
    if (!email) {
      this.router.navigate(["/auth/sign-up"]);
      return;
    }
    this.email.set(email);
  }

  async onSubmit(): Promise<void> {
    if (this.form.invalid) return;

    const { code } = this.form.getRawValue();

    try {
      await this.authService.confirmSignUp(this.email(), code);
      // 確認成功後、ログイン画面へ
      await this.router.navigate(["/auth/sign-in"], {
        queryParams: { verified: true },
      });
    } catch {
      // エラーは AuthService で処理済み
    }
  }

  async resendCode(): Promise<void> {
    this.resendSuccess.set(false);
    try {
      await this.authService.resendConfirmationCode(this.email());
      this.resendSuccess.set(true);
    } catch {
      // エラーは AuthService で処理済み
    }
  }
}
```

</details>

### タスク 3: サインインコンポーネントの実装

サインインコンポーネントを実装してください。MFA チャレンジへの対応は次章で行います。

<details>
<summary>回答</summary>

```typescript
// apps/web/src/app/auth/components/sign-in/sign-in.component.ts
import {
  Component,
  inject,
  OnInit,
  signal,
  ChangeDetectionStrategy,
} from "@angular/core";
import { CommonModule } from "@angular/common";
import { ReactiveFormsModule, FormBuilder, Validators } from "@angular/forms";
import { Router, ActivatedRoute, RouterLink } from "@angular/router";
import { AuthService } from "../../services/auth.service";

@Component({
  selector: "app-sign-in",
  standalone: true,
  imports: [CommonModule, ReactiveFormsModule, RouterLink],
  template: `
    <div class="auth-container">
      <h1>ログイン</h1>

      @if (verified()) {
      <div class="success-message">
        メールアドレスが確認されました。ログインしてください。
      </div>
      } @if (authService.error()) {
      <div class="error-message">{{ authService.error() }}</div>
      }

      <form [formGroup]="form" (ngSubmit)="onSubmit()">
        <div class="form-group">
          <label for="email">メールアドレス</label>
          <input id="email" type="email" formControlName="email" />
        </div>

        <div class="form-group">
          <label for="password">パスワード</label>
          <input id="password" type="password" formControlName="password" />
        </div>

        <button
          type="submit"
          [disabled]="form.invalid || authService.isLoading()"
        >
          @if (authService.isLoading()) {
          <span>ログイン中...</span>
          } @else {
          <span>ログイン</span>
          }
        </button>
      </form>

      <p class="auth-link">
        <a routerLink="/auth/forgot-password">パスワードをお忘れですか？</a>
      </p>
      <p class="auth-link">
        アカウントをお持ちでないですか？ <a routerLink="/auth/sign-up">登録</a>
      </p>
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class SignInComponent implements OnInit {
  readonly authService = inject(AuthService);
  private readonly fb = inject(FormBuilder);
  private readonly router = inject(Router);
  private readonly route = inject(ActivatedRoute);

  readonly verified = signal(false);

  readonly form = this.fb.nonNullable.group({
    email: ["", [Validators.required, Validators.email]],
    password: ["", [Validators.required]],
  });

  ngOnInit(): void {
    const verified = this.route.snapshot.queryParamMap.get("verified");
    this.verified.set(verified === "true");
  }

  async onSubmit(): Promise<void> {
    if (this.form.invalid) return;

    const { email, password } = this.form.getRawValue();

    try {
      const result = await this.authService.signIn(email, password);

      if (result.isSignedIn) {
        // ログイン成功、ダッシュボードへ
        await this.router.navigate(["/dashboard"]);
      } else if (result.nextStep) {
        // 追加のステップが必要（MFA など）
        switch (result.nextStep.signInStep) {
          case "CONFIRM_SIGN_UP":
            await this.router.navigate(["/auth/confirm-sign-up"], {
              queryParams: { email },
            });
            break;
          case "CONFIRM_SIGN_IN_WITH_TOTP_CODE":
            await this.router.navigate(["/auth/mfa-totp"], {
              queryParams: { email },
            });
            break;
          case "CONFIRM_SIGN_IN_WITH_SMS_CODE":
            await this.router.navigate(["/auth/mfa-sms"]);
            break;
          default:
            console.log("Unhandled sign-in step:", result.nextStep.signInStep);
        }
      }
    } catch {
      // エラーは AuthService で処理済み
    }
  }
}
```

</details>

## よくある間違い

### ❌ トークンをローカルストレージに手動保存

```typescript
// 悪い例: Amplify が自動管理するトークンを手動で保存
const session = await fetchAuthSession();
localStorage.setItem("accessToken", session.tokens?.accessToken?.toString());
```

### ✅ Amplify の自動管理を利用

```typescript
// 良い例: 必要な時に Amplify から取得
async getAccessToken(): Promise<string | undefined> {
  const session = await fetchAuthSession();
  return session.tokens?.accessToken?.toString();
}
```

### ❌ エラーハンドリングの欠如

```typescript
// 悪い例: エラーを無視
async signIn(email: string, password: string) {
  await signIn({ username: email, password });
  // エラーが発生しても何も起きない
}
```

### ✅ 適切なエラーハンドリング

```typescript
// 良い例: エラーをキャッチして適切に処理
async signIn(email: string, password: string): Promise<SignInOutput> {
  try {
    return await signIn({ username: email, password });
  } catch (error) {
    if (error instanceof Error) {
      // Cognito のエラーコードに基づいて処理
      if (error.name === 'UserNotConfirmedException') {
        // 未確認ユーザーの処理
      } else if (error.name === 'NotAuthorizedException') {
        // 認証失敗の処理
      }
    }
    throw error;
  }
}
```

## まとめ

この章では以下を学びました：

- Amplify Auth ライブラリの設定と初期化
- サインアップ、メール検証、サインインのフロー実装
- Angular Signals を使用した認証状態管理
- Reactive Forms によるフォームバリデーション

## 確認クイズ

<details>
<summary>Q1: signUp 後に nextStep.signUpStep が 'CONFIRM_SIGN_UP' を返す理由は？</summary>

**A1:**
User Pool でメール検証が有効になっているため、ユーザーはサインアップ後にメールで送信された確認コードを入力して、メールアドレスの所有を証明する必要があります。

`confirmSignUp` API を呼び出して確認コードを検証するまで、ユーザーはサインインできません。

</details>

<details>
<summary>Q2: fetchAuthSession() と getCurrentUser() の違いは？</summary>

**A2:**

- `getCurrentUser()`: 現在認証されているユーザーの基本情報（userId, username）を返す
- `fetchAuthSession()`: 現在のセッション情報（JWT トークン、認証情報）を返す

API 呼び出しにトークンが必要な場合は `fetchAuthSession()` を使用し、ユーザー情報の表示には `getCurrentUser()` を使用します。

</details>

<details>
<summary>Q3: なぜ generateSecret: false で App Client を作成したのですか？</summary>

**A3:**
SPA（Single Page Application）はブラウザ上で動作するため、クライアントシークレットを安全に保存できません。シークレットがソースコードに含まれると、誰でも閲覧可能になります。

代わりに、PKCE（Proof Key for Code Exchange）を使用した認証フローで、シークレットなしでも安全に認証を行います。

</details>

---

前の章: [01 - Cognito の基礎と環境構築](./01-cognito-basics.md)
次の章: [03 - Angular 認証 UI の構築](./03-angular-auth-ui.md)
