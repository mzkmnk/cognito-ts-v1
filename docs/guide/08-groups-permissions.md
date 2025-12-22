# 08 - グループと権限管理

## 学習の目的

- Cognito Groups の概念と使い方を理解する
- グループベースのアクセス制御（RBAC）を実装する
- JWT トークンにグループ情報を含める方法を学ぶ
- フロントエンドとバックエンドでの権限チェックを実装する

## 背景知識

### Cognito Groups とは

Cognito Groups は、ユーザーをグループ化して権限を管理する機能です：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Cognito Groups                                    │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                      User Pool                                   │   │
│  │                                                                   │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │   │
│  │  │ Admin Group │  │ Member Group│  │ Viewer Group            │ │   │
│  │  │             │  │             │  │                         │ │   │
│  │  │ • 全権限    │  │ • 読み書き  │  │ • 読み取りのみ          │ │   │
│  │  │ • ユーザー  │  │ • タスク    │  │                         │ │   │
│  │  │   管理      │  │   管理      │  │                         │ │   │
│  │  │             │  │             │  │                         │ │   │
│  │  │ User A      │  │ User B      │  │ User D                  │ │   │
│  │  │ User B      │  │ User C      │  │ User E                  │ │   │
│  │  └─────────────┘  └─────────────┘  └─────────────────────────┘ │   │
│  │                                                                   │   │
│  │  ※ ユーザーは複数のグループに所属可能                            │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### JWT トークンとグループ

グループ情報は JWT トークンの `cognito:groups` クレームに含まれます：

```json
{
  "sub": "12345678-1234-1234-1234-123456789012",
  "cognito:groups": ["Admin", "Member"],
  "email": "user@example.com",
  "token_use": "id"
}
```

### RBAC（Role-Based Access Control）

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        RBAC モデル                                       │
│                                                                          │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────────────────┐ │
│  │ User    │───▶│ Role    │───▶│ Permission│───▶│ Resource           │ │
│  │         │    │ (Group) │    │          │    │                     │ │
│  └─────────┘    └─────────┘    └─────────┘    └─────────────────────┘ │
│                                                                          │
│  例:                                                                     │
│  User A ──▶ Admin ──▶ users:write ──▶ /api/users                       │
│  User B ──▶ Member ──▶ tasks:write ──▶ /api/tasks                      │
│  User C ──▶ Viewer ──▶ tasks:read ──▶ /api/tasks (GET only)            │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

## 概念の説明

### グループの優先度

Cognito Groups には優先度（Precedence）を設定できます。ユーザーが複数のグループに所属する場合、最も優先度の高いグループの IAM ロールが適用されます：

```typescript
// 優先度: 数値が小さいほど高い
Admin: precedence = 1; // 最高優先度
Member: precedence = 2;
Viewer: precedence = 3;
```

### 権限チェックの実装パターン

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     権限チェックの実装箇所                               │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ フロントエンド                                                   │   │
│  │ • UI の表示/非表示                                               │   │
│  │ • ルートガード                                                   │   │
│  │ • ボタンの有効/無効                                              │   │
│  │ ※ セキュリティ目的ではなく UX 向上のため                         │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ バックエンド（必須）                                             │   │
│  │ • API エンドポイントの保護                                       │   │
│  │ • ビジネスロジックでの権限チェック                               │   │
│  │ • データアクセス制御                                             │   │
│  │ ※ 実際のセキュリティはここで担保                                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

## コードサンプル

### CDK でのグループ作成

```typescript
// infra/lib/cognito-stack.ts（グループ設定を追加）
import * as cdk from "aws-cdk-lib";
import * as cognito from "aws-cdk-lib/aws-cognito";
import { Construct } from "constructs";

export class CognitoStack extends cdk.Stack {
  public readonly userPool: cognito.UserPool;
  public readonly adminGroup: cognito.CfnUserPoolGroup;
  public readonly memberGroup: cognito.CfnUserPoolGroup;

  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // User Pool（既存）
    this.userPool = new cognito.UserPool(this, "TaskFlowUserPool", {
      // ... 既存の設定 ...
    });

    // Admin グループ
    this.adminGroup = new cognito.CfnUserPoolGroup(this, "AdminGroup", {
      userPoolId: this.userPool.userPoolId,
      groupName: "Admin",
      description: "管理者グループ - 全権限",
      precedence: 1,
    });

    // Member グループ
    this.memberGroup = new cognito.CfnUserPoolGroup(this, "MemberGroup", {
      userPoolId: this.userPool.userPoolId,
      groupName: "Member",
      description: "メンバーグループ - タスク管理権限",
      precedence: 2,
    });

    // Viewer グループ
    new cognito.CfnUserPoolGroup(this, "ViewerGroup", {
      userPoolId: this.userPool.userPoolId,
      groupName: "Viewer",
      description: "閲覧者グループ - 読み取り専用",
      precedence: 3,
    });
  }
}
```

### バックエンドでの権限チェック

```typescript
// apps/api/src/middleware/require-group.ts
import { createMiddleware } from "hono/factory";
import { HTTPException } from "hono/http-exception";

type Variables = {
  userId: string;
  groups: string[];
};

/**
 * 指定されたグループのいずれかに所属していることを要求
 */
export function requireGroup(...allowedGroups: string[]) {
  return createMiddleware<{ Variables: Variables }>(async (c, next) => {
    const userGroups = c.get("groups") || [];

    const hasAccess = allowedGroups.some((group) => userGroups.includes(group));

    if (!hasAccess) {
      throw new HTTPException(403, {
        message: `Access denied. Required groups: ${allowedGroups.join(
          " or "
        )}`,
      });
    }

    await next();
  });
}

/**
 * 指定されたグループの全てに所属していることを要求
 */
export function requireAllGroups(...requiredGroups: string[]) {
  return createMiddleware<{ Variables: Variables }>(async (c, next) => {
    const userGroups = c.get("groups") || [];

    const hasAllGroups = requiredGroups.every((group) =>
      userGroups.includes(group)
    );

    if (!hasAllGroups) {
      throw new HTTPException(403, {
        message: `Access denied. Required all groups: ${requiredGroups.join(
          " and "
        )}`,
      });
    }

    await next();
  });
}

/**
 * Admin グループを要求するショートカット
 */
export const requireAdmin = requireGroup("Admin");

/**
 * Admin または Member グループを要求するショートカット
 */
export const requireMember = requireGroup("Admin", "Member");
```

### 権限ベースの API ルーティング

```typescript
// apps/api/src/routes/admin.ts
import { Hono } from "hono";
import { requireAdmin } from "../middleware/require-group";

type Variables = {
  userId: string;
  groups: string[];
};

export const adminRouter = new Hono<{ Variables: Variables }>();

// 全ルートに Admin 権限を要求
adminRouter.use("*", requireAdmin);

// ユーザー一覧取得
adminRouter.get("/users", async (c) => {
  // TODO: Cognito からユーザー一覧を取得
  return c.json({ users: [] });
});

// ユーザーをグループに追加
adminRouter.post("/users/:userId/groups/:groupName", async (c) => {
  const { userId, groupName } = c.req.param();
  // TODO: Cognito API でグループに追加
  return c.json({ message: `User ${userId} added to ${groupName}` });
});

// ユーザーをグループから削除
adminRouter.delete("/users/:userId/groups/:groupName", async (c) => {
  const { userId, groupName } = c.req.param();
  // TODO: Cognito API でグループから削除
  return c.json({ message: `User ${userId} removed from ${groupName}` });
});
```

```typescript
// apps/api/src/routes/tasks.ts（権限チェックを追加）
import { Hono } from "hono";
import { requireMember, requireGroup } from "../middleware/require-group";

type Variables = {
  userId: string;
  groups: string[];
};

export const tasksRouter = new Hono<{ Variables: Variables }>();

// 読み取りは全員可能
tasksRouter.get("/", async (c) => {
  return c.json({ tasks: [] });
});

tasksRouter.get("/:id", async (c) => {
  const taskId = c.req.param("id");
  return c.json({ task: { id: taskId } });
});

// 作成・更新・削除は Member 以上
tasksRouter.post("/", requireMember, async (c) => {
  // タスク作成
  return c.json({ task: {} }, 201);
});

tasksRouter.patch("/:id", requireMember, async (c) => {
  // タスク更新
  return c.json({ task: {} });
});

tasksRouter.delete("/:id", requireMember, async (c) => {
  // タスク削除
  return c.json({ message: "Deleted" });
});
```

### フロントエンドでの権限チェック

```typescript
// apps/web/src/app/auth/services/permission.service.ts
import { Injectable, inject, computed } from "@angular/core";
import { AuthService } from "./auth.service";

export type Permission =
  | "users:read"
  | "users:write"
  | "tasks:read"
  | "tasks:write"
  | "orgs:read"
  | "orgs:write";

const ROLE_PERMISSIONS: Record<string, Permission[]> = {
  Admin: [
    "users:read",
    "users:write",
    "tasks:read",
    "tasks:write",
    "orgs:read",
    "orgs:write",
  ],
  Member: ["tasks:read", "tasks:write", "orgs:read"],
  Viewer: ["tasks:read", "orgs:read"],
};

@Injectable({
  providedIn: "root",
})
export class PermissionService {
  private readonly authService = inject(AuthService);

  /**
   * ユーザーの全権限を取得
   */
  readonly permissions = computed(() => {
    const groups = this.authService.user()?.groups || [];
    const permissions = new Set<Permission>();

    for (const group of groups) {
      const groupPermissions = ROLE_PERMISSIONS[group] || [];
      groupPermissions.forEach((p) => permissions.add(p));
    }

    return Array.from(permissions);
  });

  /**
   * 特定の権限を持っているか確認
   */
  hasPermission(permission: Permission): boolean {
    return this.permissions().includes(permission);
  }

  /**
   * いずれかの権限を持っているか確認
   */
  hasAnyPermission(...permissions: Permission[]): boolean {
    return permissions.some((p) => this.hasPermission(p));
  }

  /**
   * 全ての権限を持っているか確認
   */
  hasAllPermissions(...permissions: Permission[]): boolean {
    return permissions.every((p) => this.hasPermission(p));
  }

  /**
   * Admin グループに所属しているか
   */
  readonly isAdmin = computed(() => {
    const groups = this.authService.user()?.groups || [];
    return groups.includes("Admin");
  });

  /**
   * Member 以上の権限を持っているか
   */
  readonly isMemberOrAbove = computed(() => {
    const groups = this.authService.user()?.groups || [];
    return groups.includes("Admin") || groups.includes("Member");
  });
}
```

### 権限ベースの Guard

```typescript
// apps/web/src/app/auth/guards/permission.guard.ts
import { inject } from "@angular/core";
import { Router, type CanActivateFn } from "@angular/router";
import {
  PermissionService,
  type Permission,
} from "../services/permission.service";

export function requirePermission(...permissions: Permission[]): CanActivateFn {
  return () => {
    const permissionService = inject(PermissionService);
    const router = inject(Router);

    if (permissionService.hasAnyPermission(...permissions)) {
      return true;
    }

    // 権限がない場合は 403 ページへ
    return router.createUrlTree(["/forbidden"]);
  };
}

export function requireAdmin(): CanActivateFn {
  return () => {
    const permissionService = inject(PermissionService);
    const router = inject(Router);

    if (permissionService.isAdmin()) {
      return true;
    }

    return router.createUrlTree(["/forbidden"]);
  };
}
```

### 権限ベースの UI 表示制御

```typescript
// apps/web/src/app/shared/directives/has-permission.directive.ts
import {
  Directive,
  Input,
  TemplateRef,
  ViewContainerRef,
  inject,
  effect,
} from "@angular/core";
import {
  PermissionService,
  type Permission,
} from "../../auth/services/permission.service";

@Directive({
  selector: "[appHasPermission]",
  standalone: true,
})
export class HasPermissionDirective {
  private readonly templateRef = inject(TemplateRef<unknown>);
  private readonly viewContainer = inject(ViewContainerRef);
  private readonly permissionService = inject(PermissionService);

  private hasView = false;

  @Input() set appHasPermission(permission: Permission | Permission[]) {
    const permissions = Array.isArray(permission) ? permission : [permission];

    effect(() => {
      const hasPermission = this.permissionService.hasAnyPermission(
        ...permissions
      );

      if (hasPermission && !this.hasView) {
        this.viewContainer.createEmbeddedView(this.templateRef);
        this.hasView = true;
      } else if (!hasPermission && this.hasView) {
        this.viewContainer.clear();
        this.hasView = false;
      }
    });
  }
}

// 使用例
// <button *appHasPermission="'users:write'">ユーザー追加</button>
// <div *appHasPermission="['tasks:write', 'tasks:read']">タスク管理</div>
```

## 実装タスク

### タスク 1: 管理者用ユーザー管理画面の実装

Admin グループのユーザーのみがアクセスできるユーザー管理画面を実装してください。

<details>
<summary>回答</summary>

```typescript
// apps/web/src/app/admin/users/users.component.ts
import {
  Component,
  inject,
  signal,
  OnInit,
  ChangeDetectionStrategy,
} from "@angular/core";
import { CommonModule } from "@angular/common";
import { HttpClient } from "@angular/common/http";
import { environment } from "../../../environments/environment";

interface User {
  userId: string;
  email: string;
  groups: string[];
  createdAt: string;
}

@Component({
  selector: "app-admin-users",
  standalone: true,
  imports: [CommonModule],
  template: `
    <div class="admin-container">
      <h1>ユーザー管理</h1>

      @if (loading()) {
      <p>読み込み中...</p>
      } @else {
      <table class="users-table">
        <thead>
          <tr>
            <th>メールアドレス</th>
            <th>グループ</th>
            <th>登録日</th>
            <th>操作</th>
          </tr>
        </thead>
        <tbody>
          @for (user of users(); track user.userId) {
          <tr>
            <td>{{ user.email }}</td>
            <td>
              @for (group of user.groups; track group) {
              <span class="badge">{{ group }}</span>
              }
            </td>
            <td>{{ formatDate(user.createdAt) }}</td>
            <td>
              <button (click)="editUser(user)">編集</button>
            </td>
          </tr>
          }
        </tbody>
      </table>
      }
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class AdminUsersComponent implements OnInit {
  private readonly http = inject(HttpClient);

  readonly loading = signal(true);
  readonly users = signal<User[]>([]);

  async ngOnInit(): Promise<void> {
    this.loadUsers();
  }

  loadUsers(): void {
    this.loading.set(true);
    this.http
      .get<{ users: User[] }>(`${environment.apiUrl}/api/admin/users`)
      .subscribe({
        next: (response) => {
          this.users.set(response.users);
          this.loading.set(false);
        },
        error: () => {
          this.loading.set(false);
        },
      });
  }

  formatDate(dateString: string): string {
    return new Date(dateString).toLocaleDateString("ja-JP");
  }

  editUser(user: User): void {
    // ユーザー編集モーダルを開く
    console.log("Edit user:", user);
  }
}
```

```typescript
// apps/web/src/app/app.routes.ts に追加
{
  path: 'admin',
  canActivate: [authGuard, requireAdmin()],
  children: [
    {
      path: 'users',
      loadComponent: () =>
        import('./admin/users/users.component').then(
          (m) => m.AdminUsersComponent
        ),
    },
  ],
},
```

</details>

## よくある間違い

### ❌ フロントエンドのみで権限チェック

```typescript
// 悪い例: フロントエンドだけで権限チェック
if (this.permissionService.isAdmin()) {
  this.http.delete("/api/users/123").subscribe();
}
// バックエンドでチェックしないと、API を直接叩かれると突破される
```

### ✅ バックエンドでも必ず権限チェック

```typescript
// 良い例: バックエンドでも権限チェック
// フロントエンド
if (this.permissionService.isAdmin()) {
  this.http.delete("/api/admin/users/123").subscribe();
}

// バックエンド
adminRouter.delete("/users/:id", requireAdmin, async (c) => {
  // ここでも権限チェック済み
});
```

## まとめ

この章では以下を学びました：

- Cognito Groups の作成と管理
- JWT トークンからのグループ情報取得
- バックエンドでの権限チェックミドルウェア
- フロントエンドでの権限ベース UI 制御

## 確認クイズ

<details>
<summary>Q1: なぜフロントエンドだけでなくバックエンドでも権限チェックが必要ですか？</summary>

**A1:**
フロントエンドの権限チェックは、ブラウザの開発者ツールで簡単にバイパスできます。また、API を直接呼び出すことも可能です。

フロントエンドの権限チェックは UX 向上（不要なボタンを非表示にするなど）のためであり、実際のセキュリティはバックエンドで担保する必要があります。

</details>

<details>
<summary>Q2: グループの優先度（Precedence）は何に使われますか？</summary>

**A2:**
ユーザーが複数のグループに所属している場合、Identity Pools と連携する際に、どの IAM ロールを適用するかを決定するために使用されます。

優先度の数値が小さいグループの IAM ロールが優先的に適用されます。

</details>

---

前の章: [07 - ソーシャルログインの統合](./07-social-login.md)
次の章: [09 - Lambda Triggers によるカスタマイズ](./09-lambda-triggers.md)
