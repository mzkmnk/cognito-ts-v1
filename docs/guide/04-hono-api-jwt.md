# 04 - Hono API と JWT 検証

## 学習の目的

- Hono フレームワークの基本を理解する
- Cognito が発行した JWT トークンを検証するミドルウェアを実装する
- API Gateway + Lambda へのデプロイ方法を学ぶ
- CDK で API インフラを構築する

## 背景知識

### Hono とは

Hono は Web Standards に基づいた軽量・高速な Web フレームワークです：

- Edge Runtime（Cloudflare Workers、Deno Deploy）対応
- AWS Lambda でも動作
- Express ライクな API
- TypeScript ファーストな設計

### JWT 検証の流れ

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        JWT 検証フロー                                    │
│                                                                          │
│  ┌─────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐ │
│  │ Client  │───▶│ API Gateway │───▶│ Lambda      │───▶│ DynamoDB    │ │
│  │         │    │             │    │ (Hono)      │    │             │ │
│  └─────────┘    └─────────────┘    └─────────────┘    └─────────────┘ │
│       │               │                   │                            │
│       │               │                   │                            │
│       ▼               ▼                   ▼                            │
│  Authorization:   Cognito            JWT Middleware                    │
│  Bearer <token>   Authorizer         で再検証（任意）                   │
│                   で検証                                                │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 検証方法の選択

| 方法                           | 特徴                     | ユースケース       |
| ------------------------------ | ------------------------ | ------------------ |
| API Gateway Cognito Authorizer | マネージド、設定のみ     | シンプルな認証     |
| Lambda 内で JWT 検証           | 柔軟、カスタムロジック可 | 細かい制御が必要   |
| 両方併用                       | 二重チェック             | 高セキュリティ要件 |

## 概念の説明

### Cognito JWT の構造

```json
// ID Token のペイロード例
{
  "sub": "12345678-1234-1234-1234-123456789012",
  "email_verified": true,
  "iss": "https://cognito-idp.ap-northeast-1.amazonaws.com/ap-northeast-1_xxxxxxxx",
  "cognito:username": "user@example.com",
  "aud": "xxxxxxxxxxxxxxxxxxxxxxxxxx",
  "event_id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "token_use": "id",
  "auth_time": 1234567890,
  "exp": 1234571490,
  "iat": 1234567890,
  "email": "user@example.com",
  "custom:organizationId": "org-123"
}
```

### JWKS による検証

Cognito は公開鍵を JWKS（JSON Web Key Set）エンドポイントで公開しています：

```
https://cognito-idp.{region}.amazonaws.com/{userPoolId}/.well-known/jwks.json
```

JWT の署名を検証するには、この公開鍵を使用します。

## コードサンプル

### Hono プロジェクトのセットアップ

```bash
# apps/api ディレクトリで
cd apps/api
npm init -y
npm install hono @hono/node-server
npm install -D typescript @types/node tsx
```

```json
// apps/api/package.json
{
  "name": "taskflow-api",
  "type": "module",
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js"
  }
}
```

```json
// apps/api/tsconfig.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "outDir": "dist",
    "rootDir": "src"
  },
  "include": ["src/**/*"]
}
```

### 基本的な Hono アプリケーション

```typescript
// apps/api/src/index.ts
import { Hono } from "hono";
import { cors } from "hono/cors";
import { logger } from "hono/logger";
import { handle } from "hono/aws-lambda";
import { authMiddleware } from "./middleware/auth";
import { tasksRouter } from "./routes/tasks";
import { orgsRouter } from "./routes/orgs";

// 環境変数の型定義
type Bindings = {
  COGNITO_USER_POOL_ID: string;
  COGNITO_CLIENT_ID: string;
  AWS_REGION: string;
};

// JWT ペイロードの型定義
type Variables = {
  userId: string;
  email: string;
  organizationId?: string;
  groups: string[];
};

const app = new Hono<{ Bindings: Bindings; Variables: Variables }>();

// グローバルミドルウェア
app.use("*", logger());
app.use(
  "*",
  cors({
    origin: ["http://localhost:4200", "https://taskflow.example.com"],
    credentials: true,
  })
);

// ヘルスチェック（認証不要）
app.get("/health", (c) => c.json({ status: "ok" }));

// 認証が必要なルート
app.use("/api/*", authMiddleware);

// ルーターのマウント
app.route("/api/tasks", tasksRouter);
app.route("/api/orgs", orgsRouter);

// ユーザー情報取得エンドポイント
app.get("/api/me", (c) => {
  return c.json({
    userId: c.get("userId"),
    email: c.get("email"),
    organizationId: c.get("organizationId"),
    groups: c.get("groups"),
  });
});

// ローカル開発用
if (process.env.NODE_ENV !== "production") {
  const { serve } = await import("@hono/node-server");
  serve({ fetch: app.fetch, port: 3000 });
  console.log("Server running on http://localhost:3000");
}

// Lambda ハンドラー
export const handler = handle(app);
export default app;
```

### JWT 検証ミドルウェア

```typescript
// apps/api/src/middleware/auth.ts
import { createMiddleware } from "hono/factory";
import { HTTPException } from "hono/http-exception";
import { createRemoteJWKSet, jwtVerify, type JWTPayload } from "jose";

interface CognitoJWTPayload extends JWTPayload {
  sub: string;
  email?: string;
  "cognito:username"?: string;
  "cognito:groups"?: string[];
  "custom:organizationId"?: string;
  token_use: "id" | "access";
}

// JWKS のキャッシュ
let jwks: ReturnType<typeof createRemoteJWKSet> | null = null;

function getJWKS(region: string, userPoolId: string) {
  if (!jwks) {
    const jwksUri = `https://cognito-idp.${region}.amazonaws.com/${userPoolId}/.well-known/jwks.json`;
    jwks = createRemoteJWKSet(new URL(jwksUri));
  }
  return jwks;
}

export const authMiddleware = createMiddleware<{
  Bindings: {
    COGNITO_USER_POOL_ID: string;
    COGNITO_CLIENT_ID: string;
    AWS_REGION: string;
  };
  Variables: {
    userId: string;
    email: string;
    organizationId?: string;
    groups: string[];
  };
}>(async (c, next) => {
  const authHeader = c.req.header("Authorization");

  if (!authHeader || !authHeader.startsWith("Bearer ")) {
    throw new HTTPException(401, {
      message: "Authorization header is required",
    });
  }

  const token = authHeader.substring(7);
  const region = c.env.AWS_REGION || process.env.AWS_REGION || "ap-northeast-1";
  const userPoolId =
    c.env.COGNITO_USER_POOL_ID || process.env.COGNITO_USER_POOL_ID;
  const clientId = c.env.COGNITO_CLIENT_ID || process.env.COGNITO_CLIENT_ID;

  if (!userPoolId || !clientId) {
    throw new HTTPException(500, {
      message: "Cognito configuration is missing",
    });
  }

  try {
    const jwksClient = getJWKS(region, userPoolId);
    const issuer = `https://cognito-idp.${region}.amazonaws.com/${userPoolId}`;

    const { payload } = await jwtVerify(token, jwksClient, {
      issuer,
      audience: clientId,
    });

    const cognitoPayload = payload as CognitoJWTPayload;

    // トークンタイプの検証（ID トークンを期待）
    if (cognitoPayload.token_use !== "id") {
      throw new HTTPException(401, { message: "Invalid token type" });
    }

    // コンテキストにユーザー情報を設定
    c.set("userId", cognitoPayload.sub);
    c.set(
      "email",
      cognitoPayload.email || cognitoPayload["cognito:username"] || ""
    );
    c.set("organizationId", cognitoPayload["custom:organizationId"]);
    c.set("groups", cognitoPayload["cognito:groups"] || []);

    await next();
  } catch (error) {
    if (error instanceof HTTPException) {
      throw error;
    }
    console.error("JWT verification failed:", error);
    throw new HTTPException(401, { message: "Invalid or expired token" });
  }
});
```

### タスク API ルーター

```typescript
// apps/api/src/routes/tasks.ts
import { Hono } from "hono";
import { zValidator } from "@hono/zod-validator";
import { z } from "zod";

type Variables = {
  userId: string;
  email: string;
  organizationId?: string;
  groups: string[];
};

export const tasksRouter = new Hono<{ Variables: Variables }>();

// バリデーションスキーマ
const createTaskSchema = z.object({
  title: z.string().min(1).max(200),
  description: z.string().max(1000).optional(),
  assigneeId: z.string().optional(),
  dueDate: z.string().datetime().optional(),
});

const updateTaskSchema = createTaskSchema.partial();

// タスク一覧取得
tasksRouter.get("/", async (c) => {
  const userId = c.get("userId");
  const organizationId = c.get("organizationId");

  // TODO: DynamoDB からタスクを取得
  const tasks = [
    {
      id: "task-1",
      title: "サンプルタスク",
      description: "これはサンプルです",
      status: "pending",
      createdBy: userId,
      organizationId,
      createdAt: new Date().toISOString(),
    },
  ];

  return c.json({ tasks });
});

// タスク作成
tasksRouter.post("/", zValidator("json", createTaskSchema), async (c) => {
  const userId = c.get("userId");
  const organizationId = c.get("organizationId");
  const body = c.req.valid("json");

  if (!organizationId) {
    return c.json({ error: "Organization is required" }, 400);
  }

  // TODO: DynamoDB にタスクを保存
  const task = {
    id: `task-${Date.now()}`,
    ...body,
    status: "pending",
    createdBy: userId,
    organizationId,
    createdAt: new Date().toISOString(),
  };

  return c.json({ task }, 201);
});

// タスク更新
tasksRouter.patch("/:id", zValidator("json", updateTaskSchema), async (c) => {
  const taskId = c.req.param("id");
  const userId = c.get("userId");
  const body = c.req.valid("json");

  // TODO: DynamoDB でタスクを更新
  const task = {
    id: taskId,
    ...body,
    updatedBy: userId,
    updatedAt: new Date().toISOString(),
  };

  return c.json({ task });
});

// タスク削除
tasksRouter.delete("/:id", async (c) => {
  const taskId = c.req.param("id");

  // TODO: DynamoDB からタスクを削除

  return c.json({ message: "Task deleted", id: taskId });
});
```

### CDK による API インフラ構築

```typescript
// infra/lib/api-stack.ts
import * as cdk from "aws-cdk-lib";
import * as lambda from "aws-cdk-lib/aws-lambda";
import * as apigateway from "aws-cdk-lib/aws-apigateway";
import * as cognito from "aws-cdk-lib/aws-cognito";
import { Construct } from "constructs";
import * as path from "path";

interface ApiStackProps extends cdk.StackProps {
  userPool: cognito.IUserPool;
  userPoolClient: cognito.IUserPoolClient;
}

export class ApiStack extends cdk.Stack {
  public readonly api: apigateway.RestApi;

  constructor(scope: Construct, id: string, props: ApiStackProps) {
    super(scope, id, props);

    // Lambda 関数
    const apiFunction = new lambda.Function(this, "ApiFunction", {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: "index.handler",
      code: lambda.Code.fromAsset(path.join(__dirname, "../../apps/api/dist")),
      environment: {
        COGNITO_USER_POOL_ID: props.userPool.userPoolId,
        COGNITO_CLIENT_ID: props.userPoolClient.userPoolClientId,
        AWS_REGION: this.region,
        NODE_ENV: "production",
      },
      timeout: cdk.Duration.seconds(30),
      memorySize: 256,
    });

    // API Gateway
    this.api = new apigateway.RestApi(this, "TaskFlowApi", {
      restApiName: "TaskFlow API",
      description: "TaskFlow REST API",
      deployOptions: {
        stageName: "v1",
      },
      defaultCorsPreflightOptions: {
        allowOrigins: apigateway.Cors.ALL_ORIGINS,
        allowMethods: apigateway.Cors.ALL_METHODS,
        allowHeaders: ["Content-Type", "Authorization"],
      },
    });

    // Cognito Authorizer
    const authorizer = new apigateway.CognitoUserPoolsAuthorizer(
      this,
      "CognitoAuthorizer",
      {
        cognitoUserPools: [props.userPool],
        identitySource: "method.request.header.Authorization",
      }
    );

    // Lambda 統合
    const lambdaIntegration = new apigateway.LambdaIntegration(apiFunction);

    // ヘルスチェック（認証不要）
    const healthResource = this.api.root.addResource("health");
    healthResource.addMethod("GET", lambdaIntegration);

    // API リソース（認証必要）
    const apiResource = this.api.root.addResource("api");

    // プロキシリソースで全てのパスを Lambda に転送
    const proxyResource = apiResource.addProxy({
      defaultIntegration: lambdaIntegration,
      defaultMethodOptions: {
        authorizer,
        authorizationType: apigateway.AuthorizationType.COGNITO,
      },
    });

    // 出力
    new cdk.CfnOutput(this, "ApiEndpoint", {
      value: this.api.url,
      description: "API Gateway endpoint URL",
    });
  }
}
```

## 実装タスク

### タスク 1: グループベースの認可ミドルウェア

特定のグループ（Admin など）のみがアクセスできるミドルウェアを実装してください。

<details>
<summary>ヒント</summary>

JWT ペイロードの `cognito:groups` クレームにユーザーが所属するグループが含まれています。

</details>

<details>
<summary>回答</summary>

```typescript
// apps/api/src/middleware/require-group.ts
import { createMiddleware } from "hono/factory";
import { HTTPException } from "hono/http-exception";

type Variables = {
  groups: string[];
};

export function requireGroup(...allowedGroups: string[]) {
  return createMiddleware<{ Variables: Variables }>(async (c, next) => {
    const userGroups = c.get("groups") || [];

    const hasAccess = allowedGroups.some((group) => userGroups.includes(group));

    if (!hasAccess) {
      throw new HTTPException(403, {
        message: `Access denied. Required groups: ${allowedGroups.join(", ")}`,
      });
    }

    await next();
  });
}

// 使用例
// apps/api/src/routes/admin.ts
import { Hono } from "hono";
import { requireGroup } from "../middleware/require-group";

export const adminRouter = new Hono();

// Admin グループのみアクセス可能
adminRouter.use("*", requireGroup("Admin"));

adminRouter.get("/users", async (c) => {
  // Admin のみがアクセスできる
  return c.json({ users: [] });
});

adminRouter.delete("/users/:id", async (c) => {
  const userId = c.req.param("id");
  // ユーザー削除処理
  return c.json({ message: "User deleted", id: userId });
});
```

</details>

### タスク 2: エラーハンドリングミドルウェア

統一されたエラーレスポンスを返すミドルウェアを実装してください。

<details>
<summary>回答</summary>

```typescript
// apps/api/src/middleware/error-handler.ts
import { ErrorHandler } from "hono";
import { HTTPException } from "hono/http-exception";
import { ZodError } from "zod";

interface ErrorResponse {
  error: {
    message: string;
    code: string;
    details?: unknown;
  };
}

export const errorHandler: ErrorHandler = (err, c) => {
  console.error("Error:", err);

  // Zod バリデーションエラー
  if (err instanceof ZodError) {
    const response: ErrorResponse = {
      error: {
        message: "Validation failed",
        code: "VALIDATION_ERROR",
        details: err.errors.map((e) => ({
          path: e.path.join("."),
          message: e.message,
        })),
      },
    };
    return c.json(response, 400);
  }

  // HTTP 例外
  if (err instanceof HTTPException) {
    const response: ErrorResponse = {
      error: {
        message: err.message,
        code: `HTTP_${err.status}`,
      },
    };
    return c.json(response, err.status);
  }

  // その他のエラー
  const response: ErrorResponse = {
    error: {
      message: "Internal server error",
      code: "INTERNAL_ERROR",
    },
  };
  return c.json(response, 500);
};

// 使用例
// apps/api/src/index.ts
app.onError(errorHandler);
```

</details>

## よくある間違い

### ❌ JWKS を毎回フェッチ

```typescript
// 悪い例: リクエストごとに JWKS をフェッチ
const jwksUri = `https://cognito-idp.${region}.amazonaws.com/${userPoolId}/.well-known/jwks.json`;
const jwks = createRemoteJWKSet(new URL(jwksUri)); // 毎回作成
```

### ✅ JWKS をキャッシュ

```typescript
// 良い例: JWKS をキャッシュ
let jwks: ReturnType<typeof createRemoteJWKSet> | null = null;

function getJWKS(region: string, userPoolId: string) {
  if (!jwks) {
    const jwksUri = `https://cognito-idp.${region}.amazonaws.com/${userPoolId}/.well-known/jwks.json`;
    jwks = createRemoteJWKSet(new URL(jwksUri));
  }
  return jwks;
}
```

### ❌ トークンタイプを検証しない

```typescript
// 悪い例: Access Token を ID Token として使用してしまう
const { payload } = await jwtVerify(token, jwks);
c.set("email", payload.email); // Access Token には email がない場合がある
```

### ✅ トークンタイプを検証

```typescript
// 良い例: トークンタイプを確認
if (payload.token_use !== "id") {
  throw new HTTPException(401, { message: "Invalid token type" });
}
```

## まとめ

この章では以下を学びました：

- Hono フレームワークの基本構造
- Cognito JWT の検証方法（JWKS 使用）
- グループベースの認可実装
- CDK による API Gateway + Lambda のデプロイ

## 確認クイズ

<details>
<summary>Q1: ID Token と Access Token の違いは何ですか？</summary>

**A1:**

- ID Token: ユーザーの属性情報（email, name など）を含む。フロントエンドでユーザー情報を表示する際に使用
- Access Token: API へのアクセス権限を証明。スコープ情報を含む。バックエンド API の認証に使用

一般的に、API 認証には Access Token を使用しますが、ユーザー属性が必要な場合は ID Token を使用します。

</details>

<details>
<summary>Q2: なぜ JWKS をキャッシュすべきですか？</summary>

**A2:**
JWKS エンドポイントへのリクエストはネットワーク遅延を伴います。毎回フェッチすると：

- レスポンス時間が増加
- Cognito への不要なリクエストが発生
- レート制限に達する可能性

JWKS は頻繁に変更されないため、キャッシュしても問題ありません。`jose` ライブラリの `createRemoteJWKSet` は内部でキャッシュを管理します。

</details>

---

前の章: [03 - Angular 認証 UI の構築](./03-angular-auth-ui.md)
次の章: [05 - MFA（多要素認証）の実装](./05-mfa-implementation.md)
