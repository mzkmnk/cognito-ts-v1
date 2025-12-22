# 03 - Angular 認証 UI の構築

## 学習の目的

- Angular の AuthGuard を実装して認証が必要なルートを保護する
- HTTP Interceptor でリクエストに認証トークンを自動付与する
- 認証状態に応じた UI の切り替えを実装する
- ルーティング設計のベストプラクティスを学ぶ

## 背景知識

### Angular の認証パターン

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     Angular 認証アーキテクチャ                           │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                        Router                                    │   │
│  │  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────┐ │   │
│  │  │ Public      │    │ Protected   │    │ Admin               │ │   │
│  │  │ Routes      │    │ Routes      │    │ Routes              │ │   │
│  │  │             │    │             │    │                     │ │   │
│  │  │ /auth/*     │    │ /dashboard  │    │ /admin/*            │ │   │
│  │  │ /landing    │    │ /tasks      │    │                     │ │   │
│  │  └─────────────┘    └──────┬──────┘    └──────────┬──────────┘ │   │
│  │                            │                      │             │   │
│  │                     ┌──────▼──────┐        ┌──────▼──────┐     │   │
│  │                     │ authGuard   │        │ adminGuard  │     │   │
│  │                     └──────┬──────┘        └──────┬──────┘     │   │
│  │                            │                      │             │   │
│  └────────────────────────────┼──────────────────────┼─────────────┘   │
│                               │                      │                  │
│                        ┌──────▼──────────────────────▼──────┐          │
│                        │           AuthService              │          │
│                        │  • isAuthenticated()               │          │
│                        │  • user()                          │          │
│                        │  • getAccessToken()                │          │
│                        └──────────────────────────────────────┘          │
│                                        │                                │
│                        ┌───────────────▼───────────────┐               │
│                        │      HTTP Interceptor         │               │
│                        │  Authorization: Bearer <token>│               │
│                        └───────────────────────────────┘               │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

## 概念の説明

### Functional Guards（Angular 14+）

Angular 14 以降では、クラスベースの Guard よりも関数ベースの Guard が推奨されています：

```typescript
// 関数ベースの Guard（推奨）
export const authGuard: CanActivateFn = () => {
  const authService = inject(AuthService);
  const router = inject(Router);

  if (authService.isAuthenticated()) {
    return true;
  }
  return router.createUrlTree(["/auth/sign-in"]);
};
```

### HTTP Interceptor

すべての HTTP リクエストに認証トークンを自動付与します：

```typescript
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  // トークンを取得してヘッダーに追加
  const token = getAccessToken();
  if (token) {
    req = req.clone({
      setHeaders: { Authorization: `Bearer ${token}` },
    });
  }
  return next(req);
};
```

## コードサンプル

### Auth Guard の実装

```typescript
// apps/web/src/app/auth/guards/auth.guard.ts
import { inject } from "@angular/core";
import { Router, type CanActivateFn } from "@angular/router";
import { AuthService } from "../services/auth.service";

export const authGuard: CanActivateFn = async () => {
  const authService = inject(AuthService);
  const router = inject(Router);

  // 認証状態の確認を待つ
  if (authService.isLoading()) {
    await new Promise<void>((resolve) => {
      const checkLoading = setInterval(() => {
        if (!authService.isLoading()) {
          clearInterval(checkLoading);
          resolve();
        }
      }, 50);
    });
  }

  if (authService.isAuthenticated()) {
    return true;
  }

  // 未認証の場合はログイン画面へリダイレクト
  return router.createUrlTree(["/auth/sign-in"]);
};

// 認証済みユーザーが認証ページにアクセスした場合のガード
export const guestGuard: CanActivateFn = async () => {
  const authService = inject(AuthService);
  const router = inject(Router);

  if (authService.isLoading()) {
    await new Promise<void>((resolve) => {
      const checkLoading = setInterval(() => {
        if (!authService.isLoading()) {
          clearInterval(checkLoading);
          resolve();
        }
      }, 50);
    });
  }

  if (!authService.isAuthenticated()) {
    return true;
  }

  // 認証済みの場合はダッシュボードへリダイレクト
  return router.createUrlTree(["/dashboard"]);
};
```

### HTTP Interceptor の実装

```typescript
// apps/web/src/app/auth/interceptors/auth.interceptor.ts
import { HttpInterceptorFn, HttpErrorResponse } from "@angular/common/http";
import { inject } from "@angular/core";
import { Router } from "@angular/router";
import { from, switchMap, catchError, throwError } from "rxjs";
import { AuthService } from "../services/auth.service";

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const authService = inject(AuthService);
  const router = inject(Router);

  // 認証が不要なエンドポイントはスキップ
  const publicUrls = ["/auth/", "/public/"];
  if (publicUrls.some((url) => req.url.includes(url))) {
    return next(req);
  }

  return from(authService.getAccessToken()).pipe(
    switchMap((token) => {
      if (token) {
        const authReq = req.clone({
          setHeaders: {
            Authorization: `Bearer ${token}`,
          },
        });
        return next(authReq);
      }
      return next(req);
    }),
    catchError((error: HttpErrorResponse) => {
      if (error.status === 401) {
        // 認証エラーの場合はログアウトしてログイン画面へ
        authService.signOut().then(() => {
          router.navigate(["/auth/sign-in"]);
        });
      }
      return throwError(() => error);
    })
  );
};
```

### ルーティング設定

```typescript
// apps/web/src/app/app.routes.ts
import { Routes } from "@angular/router";
import { authGuard, guestGuard } from "./auth/guards/auth.guard";

export const routes: Routes = [
  // 公開ルート
  {
    path: "",
    redirectTo: "dashboard",
    pathMatch: "full",
  },

  // 認証ルート（ゲストのみ）
  {
    path: "auth",
    canActivate: [guestGuard],
    children: [
      {
        path: "sign-in",
        loadComponent: () =>
          import("./auth/components/sign-in/sign-in.component").then(
            (m) => m.SignInComponent
          ),
      },
      {
        path: "sign-up",
        loadComponent: () =>
          import("./auth/components/sign-up/sign-up.component").then(
            (m) => m.SignUpComponent
          ),
      },
      {
        path: "confirm-sign-up",
        loadComponent: () =>
          import(
            "./auth/components/confirm-sign-up/confirm-sign-up.component"
          ).then((m) => m.ConfirmSignUpComponent),
      },
      {
        path: "forgot-password",
        loadComponent: () =>
          import(
            "./auth/components/forgot-password/forgot-password.component"
          ).then((m) => m.ForgotPasswordComponent),
      },
      {
        path: "",
        redirectTo: "sign-in",
        pathMatch: "full",
      },
    ],
  },

  // 保護されたルート（認証必須）
  {
    path: "dashboard",
    canActivate: [authGuard],
    loadComponent: () =>
      import("./dashboard/dashboard.component").then(
        (m) => m.DashboardComponent
      ),
  },
  {
    path: "tasks",
    canActivate: [authGuard],
    loadComponent: () =>
      import("./tasks/tasks.component").then((m) => m.TasksComponent),
  },

  // 404
  {
    path: "**",
    redirectTo: "dashboard",
  },
];
```

### ヘッダーコンポーネント（認証状態表示）

```typescript
// apps/web/src/app/shared/components/header/header.component.ts
import { Component, inject, ChangeDetectionStrategy } from "@angular/core";
import { CommonModule } from "@angular/common";
import { RouterLink } from "@angular/router";
import { AuthService } from "../../../auth/services/auth.service";

@Component({
  selector: "app-header",
  standalone: true,
  imports: [CommonModule, RouterLink],
  template: `
    <header class="header">
      <div class="logo">
        <a routerLink="/">TaskFlow</a>
      </div>

      <nav class="nav">
        @if (authService.isAuthenticated()) {
        <a routerLink="/dashboard">ダッシュボード</a>
        <a routerLink="/tasks">タスク</a>

        <div class="user-menu">
          <span class="user-email">{{ authService.user()?.email }}</span>
          <button (click)="onSignOut()" class="sign-out-btn">ログアウト</button>
        </div>
        } @else {
        <a routerLink="/auth/sign-in">ログイン</a>
        <a routerLink="/auth/sign-up" class="btn-primary">登録</a>
        }
      </nav>
    </header>
  `,
  styles: [
    `
      .header {
        display: flex;
        justify-content: space-between;
        align-items: center;
        padding: 1rem 2rem;
        background: #fff;
        box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
      }
      .nav {
        display: flex;
        gap: 1rem;
        align-items: center;
      }
      .user-menu {
        display: flex;
        align-items: center;
        gap: 1rem;
      }
      .user-email {
        color: #666;
      }
      .sign-out-btn {
        background: none;
        border: 1px solid #ddd;
        padding: 0.5rem 1rem;
        border-radius: 4px;
        cursor: pointer;
      }
    `,
  ],
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class HeaderComponent {
  readonly authService = inject(AuthService);

  async onSignOut(): Promise<void> {
    await this.authService.signOut();
  }
}
```

## 実装タスク

### タスク 1: ダッシュボードコンポーネントの実装

認証済みユーザー向けのダッシュボードを実装してください。

**要件:**

- ユーザー情報の表示（メールアドレス）
- 認証状態の確認
- ログアウトボタン

<details>
<summary>回答</summary>

```typescript
// apps/web/src/app/dashboard/dashboard.component.ts
import { Component, inject, ChangeDetectionStrategy } from "@angular/core";
import { CommonModule } from "@angular/common";
import { RouterLink } from "@angular/router";
import { AuthService } from "../auth/services/auth.service";

@Component({
  selector: "app-dashboard",
  standalone: true,
  imports: [CommonModule, RouterLink],
  template: `
    <div class="dashboard">
      <h1>ダッシュボード</h1>

      @if (authService.user(); as user) {
      <div class="user-info">
        <h2>ようこそ、{{ user.email }} さん</h2>
        <p>ユーザーID: {{ user.userId }}</p>
        <p>メール確認: {{ user.emailVerified ? "完了" : "未完了" }}</p>
      </div>
      }

      <div class="quick-actions">
        <h3>クイックアクション</h3>
        <div class="action-cards">
          <a routerLink="/tasks" class="action-card">
            <span class="icon">📋</span>
            <span>タスク管理</span>
          </a>
          <a routerLink="/settings/security" class="action-card">
            <span class="icon">🔐</span>
            <span>セキュリティ設定</span>
          </a>
        </div>
      </div>
    </div>
  `,
  styles: [
    `
      .dashboard {
        padding: 2rem;
        max-width: 1200px;
        margin: 0 auto;
      }
      .user-info {
        background: #f5f5f5;
        padding: 1.5rem;
        border-radius: 8px;
        margin-bottom: 2rem;
      }
      .action-cards {
        display: grid;
        grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
        gap: 1rem;
      }
      .action-card {
        display: flex;
        flex-direction: column;
        align-items: center;
        padding: 2rem;
        background: #fff;
        border: 1px solid #ddd;
        border-radius: 8px;
        text-decoration: none;
        color: inherit;
        transition: box-shadow 0.2s;
      }
      .action-card:hover {
        box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
      }
      .icon {
        font-size: 2rem;
        margin-bottom: 0.5rem;
      }
    `,
  ],
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class DashboardComponent {
  readonly authService = inject(AuthService);
}
```

</details>

### タスク 2: パスワードリセット機能の実装

パスワードを忘れた場合のリセット機能を実装してください。

<details>
<summary>ヒント</summary>

Amplify Auth の `resetPassword` と `confirmResetPassword` を使用します。
フローは以下の通りです：

1. メールアドレスを入力して `resetPassword` を呼び出す
2. メールで確認コードを受信
3. 確認コードと新しいパスワードで `confirmResetPassword` を呼び出す

</details>

<details>
<summary>回答</summary>

```typescript
// apps/web/src/app/auth/services/auth.service.ts に追加
import { resetPassword, confirmResetPassword } from 'aws-amplify/auth';

// AuthService クラスに追加
async resetPassword(email: string): Promise<void> {
  this._isLoading.set(true);
  this._error.set(null);

  try {
    await resetPassword({ username: email });
  } catch (error) {
    const message = error instanceof Error ? error.message : 'パスワードリセットに失敗しました';
    this._error.set(message);
    throw error;
  } finally {
    this._isLoading.set(false);
  }
}

async confirmResetPassword(
  email: string,
  code: string,
  newPassword: string
): Promise<void> {
  this._isLoading.set(true);
  this._error.set(null);

  try {
    await confirmResetPassword({
      username: email,
      confirmationCode: code,
      newPassword,
    });
  } catch (error) {
    const message = error instanceof Error ? error.message : 'パスワードの変更に失敗しました';
    this._error.set(message);
    throw error;
  } finally {
    this._isLoading.set(false);
  }
}
```

```typescript
// apps/web/src/app/auth/components/forgot-password/forgot-password.component.ts
import {
  Component,
  inject,
  signal,
  ChangeDetectionStrategy,
} from "@angular/core";
import { CommonModule } from "@angular/common";
import { ReactiveFormsModule, FormBuilder, Validators } from "@angular/forms";
import { Router, RouterLink } from "@angular/router";
import { AuthService } from "../../services/auth.service";

@Component({
  selector: "app-forgot-password",
  standalone: true,
  imports: [CommonModule, ReactiveFormsModule, RouterLink],
  template: `
    <div class="auth-container">
      <h1>パスワードリセット</h1>

      @if (authService.error()) {
      <div class="error-message">{{ authService.error() }}</div>
      } @if (!codeSent()) {
      <!-- Step 1: メールアドレス入力 -->
      <form [formGroup]="emailForm" (ngSubmit)="onRequestCode()">
        <p>登録したメールアドレスを入力してください</p>

        <div class="form-group">
          <label for="email">メールアドレス</label>
          <input id="email" type="email" formControlName="email" />
        </div>

        <button
          type="submit"
          [disabled]="emailForm.invalid || authService.isLoading()"
        >
          @if (authService.isLoading()) { 送信中... } @else { 確認コードを送信 }
        </button>
      </form>
      } @else {
      <!-- Step 2: 確認コードと新パスワード入力 -->
      <form [formGroup]="resetForm" (ngSubmit)="onResetPassword()">
        <p>
          {{
            email()
          }}
          に送信された確認コードと新しいパスワードを入力してください
        </p>

        <div class="form-group">
          <label for="code">確認コード</label>
          <input id="code" type="text" formControlName="code" maxlength="6" />
        </div>

        <div class="form-group">
          <label for="newPassword">新しいパスワード</label>
          <input
            id="newPassword"
            type="password"
            formControlName="newPassword"
          />
        </div>

        <div class="form-group">
          <label for="confirmPassword">新しいパスワード（確認）</label>
          <input
            id="confirmPassword"
            type="password"
            formControlName="confirmPassword"
          />
        </div>

        <button
          type="submit"
          [disabled]="resetForm.invalid || authService.isLoading()"
        >
          @if (authService.isLoading()) { 変更中... } @else { パスワードを変更 }
        </button>
      </form>
      }

      <p class="auth-link">
        <a routerLink="/auth/sign-in">ログイン画面に戻る</a>
      </p>
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class ForgotPasswordComponent {
  readonly authService = inject(AuthService);
  private readonly fb = inject(FormBuilder);
  private readonly router = inject(Router);

  readonly codeSent = signal(false);
  readonly email = signal("");

  readonly emailForm = this.fb.nonNullable.group({
    email: ["", [Validators.required, Validators.email]],
  });

  readonly resetForm = this.fb.nonNullable.group({
    code: ["", [Validators.required, Validators.minLength(6)]],
    newPassword: ["", [Validators.required, Validators.minLength(8)]],
    confirmPassword: ["", [Validators.required]],
  });

  async onRequestCode(): Promise<void> {
    if (this.emailForm.invalid) return;

    const { email } = this.emailForm.getRawValue();

    try {
      await this.authService.resetPassword(email);
      this.email.set(email);
      this.codeSent.set(true);
    } catch {
      // エラーは AuthService で処理済み
    }
  }

  async onResetPassword(): Promise<void> {
    if (this.resetForm.invalid) return;

    const { code, newPassword, confirmPassword } = this.resetForm.getRawValue();

    if (newPassword !== confirmPassword) {
      return;
    }

    try {
      await this.authService.confirmResetPassword(
        this.email(),
        code,
        newPassword
      );
      await this.router.navigate(["/auth/sign-in"], {
        queryParams: { passwordReset: true },
      });
    } catch {
      // エラーは AuthService で処理済み
    }
  }
}
```

</details>

## よくある間違い

### ❌ Guard で同期的に認証状態を確認

```typescript
// 悪い例: 初期化前に認証状態を確認してしまう
export const authGuard: CanActivateFn = () => {
  const authService = inject(AuthService);
  return authService.isAuthenticated(); // 初期化中は false になる
};
```

### ✅ 初期化完了を待ってから確認

```typescript
// 良い例: ローディング完了を待つ
export const authGuard: CanActivateFn = async () => {
  const authService = inject(AuthService);

  // 初期化完了を待つ
  while (authService.isLoading()) {
    await new Promise((resolve) => setTimeout(resolve, 50));
  }

  return authService.isAuthenticated();
};
```

## まとめ

この章では以下を学びました：

- Functional Guards による認証ルートの保護
- HTTP Interceptor による自動トークン付与
- 認証状態に応じた UI の切り替え
- パスワードリセット機能の実装

## 確認クイズ

<details>
<summary>Q1: guestGuard の役割は何ですか？</summary>

**A1:**
認証済みユーザーが認証ページ（ログイン、登録など）にアクセスした場合、ダッシュボードなどの適切なページにリダイレクトします。

これにより、すでにログインしているユーザーが再度ログインページを見ることを防ぎます。

</details>

<details>
<summary>Q2: HTTP Interceptor で 401 エラーを受け取った場合、どう処理すべきですか？</summary>

**A2:**
401 エラーは認証が無効（トークン期限切れなど）を示すため、以下の処理を行います：

1. ユーザーをサインアウト
2. ログイン画面にリダイレクト
3. 必要に応じて、元のページ URL を保存して再ログイン後にリダイレクト

Amplify は自動的にトークンをリフレッシュしますが、リフレッシュトークンも期限切れの場合は 401 が返されます。

</details>

---

前の章: [02 - 基本認証フローの実装](./02-basic-auth-flow.md)
次の章: [04 - Hono API と JWT 検証](./04-hono-api-jwt.md)
