# 09 - Lambda Triggers によるカスタマイズ

## 学習の目的

- Cognito Lambda Triggers の種類と用途を理解する
- Pre Sign-up トリガーでサインアップをカスタマイズする
- Post Confirmation トリガーで初期データを作成する
- Pre Token Generation トリガーでトークンをカスタマイズする

## 背景知識

### Lambda Triggers とは

Lambda Triggers は、Cognito の認証フローの各段階で Lambda 関数を呼び出し、デフォルトの動作をカスタマイズする機能です：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     Lambda Triggers の種類                               │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ サインアップ関連                                                 │   │
│  │                                                                   │   │
│  │ Pre Sign-up ──▶ サインアップ前の検証・自動確認                   │   │
│  │ Post Confirmation ──▶ 確認後の処理（DB 登録など）                │   │
│  │ Pre Sign-up (External Provider) ──▶ フェデレーション時の処理     │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 認証関連                                                         │   │
│  │                                                                   │   │
│  │ Pre Authentication ──▶ 認証前の検証                              │   │
│  │ Post Authentication ──▶ 認証後の処理（ログ記録など）             │   │
│  │ Pre Token Generation ──▶ トークンのカスタマイズ                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ カスタム認証                                                     │   │
│  │                                                                   │   │
│  │ Define Auth Challenge ──▶ チャレンジの定義                       │   │
│  │ Create Auth Challenge ──▶ チャレンジの作成                       │   │
│  │ Verify Auth Challenge ──▶ チャレンジの検証                       │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ その他                                                           │   │
│  │                                                                   │   │
│  │ Custom Message ──▶ メッセージのカスタマイズ                      │   │
│  │ User Migration ──▶ 既存ユーザーの移行                            │   │
│  │ Custom Email/SMS Sender ──▶ カスタム送信                         │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### トリガーの実行タイミング

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     サインアップフローでのトリガー                       │
│                                                                          │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐             │
│  │ SignUp  │───▶│ Pre     │───▶│ Confirm │───▶│ Post    │             │
│  │ Request │    │ Sign-up │    │ Sign-up │    │ Confirm │             │
│  └─────────┘    └─────────┘    └─────────┘    └─────────┘             │
│                      │                             │                    │
│                      ▼                             ▼                    │
│               • ドメイン検証              • DB にユーザー作成          │
│               • 自動確認                  • ウェルカムメール           │
│               • 自動検証                  • デフォルトグループ追加     │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                     サインインフローでのトリガー                         │
│                                                                          │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐             │
│  │ SignIn  │───▶│ Pre     │───▶│ Post    │───▶│ Pre Token│             │
│  │ Request │    │ Auth    │    │ Auth    │    │ Gen     │             │
│  └─────────┘    └─────────┘    └─────────┘    └─────────┘             │
│                      │              │              │                    │
│                      ▼              ▼              ▼                    │
│               • IP 制限       • ログ記録     • カスタムクレーム        │
│               • 時間制限      • 通知送信     • グループ情報追加        │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

## 概念の説明

### イベント構造

Lambda Triggers は共通のイベント構造を持ちます：

```typescript
interface CognitoTriggerEvent {
  version: string;
  triggerSource: string; // 例: 'PreSignUp_SignUp'
  region: string;
  userPoolId: string;
  userName: string;
  callerContext: {
    awsSdkVersion: string;
    clientId: string;
  };
  request: {
    userAttributes: Record<string, string>;
    // トリガーごとに追加のプロパティ
  };
  response: {
    // トリガーごとに異なるレスポンス
  };
}
```

### トリガーソース

同じトリガーでも、呼び出し元によって `triggerSource` が異なります：

| トリガー    | triggerSource              | 説明                         |
| ----------- | -------------------------- | ---------------------------- |
| Pre Sign-up | PreSignUp_SignUp           | 通常のサインアップ           |
| Pre Sign-up | PreSignUp_AdminCreateUser  | 管理者によるユーザー作成     |
| Pre Sign-up | PreSignUp_ExternalProvider | フェデレーションサインアップ |

## コードサンプル

### Pre Sign-up トリガー

```typescript
// infra/lambda/pre-signup/index.ts
import type {
  PreSignUpTriggerEvent,
  PreSignUpTriggerHandler,
} from "aws-lambda";

// 許可するメールドメイン
const ALLOWED_DOMAINS = ["example.com", "company.co.jp"];

export const handler: PreSignUpTriggerHandler = async (event) => {
  console.log("Pre Sign-up trigger:", JSON.stringify(event, null, 2));

  const { triggerSource, request, response } = event;
  const email = request.userAttributes.email;

  // 通常のサインアップの場合のみドメインチェック
  if (triggerSource === "PreSignUp_SignUp") {
    // メールドメインの検証
    const domain = email.split("@")[1];
    if (!ALLOWED_DOMAINS.includes(domain)) {
      throw new Error(`Email domain ${domain} is not allowed`);
    }
  }

  // フェデレーションユーザーの場合は自動確認
  if (triggerSource === "PreSignUp_ExternalProvider") {
    response.autoConfirmUser = true;
    response.autoVerifyEmail = true;
  }

  // 特定のドメインは自動確認（社内ユーザーなど）
  if (email.endsWith("@company.co.jp")) {
    response.autoConfirmUser = true;
    response.autoVerifyEmail = true;
  }

  return event;
};
```

### Post Confirmation トリガー

```typescript
// infra/lambda/post-confirmation/index.ts
import type {
  PostConfirmationTriggerEvent,
  PostConfirmationTriggerHandler,
} from "aws-lambda";
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import { DynamoDBDocumentClient, PutCommand } from "@aws-sdk/lib-dynamodb";
import {
  CognitoIdentityProviderClient,
  AdminAddUserToGroupCommand,
} from "@aws-sdk/client-cognito-identity-provider";

const dynamoClient = new DynamoDBClient({});
const docClient = DynamoDBDocumentClient.from(dynamoClient);
const cognitoClient = new CognitoIdentityProviderClient({});

const USERS_TABLE = process.env.USERS_TABLE || "taskflow-users";

export const handler: PostConfirmationTriggerHandler = async (event) => {
  console.log("Post Confirmation trigger:", JSON.stringify(event, null, 2));

  const { userPoolId, userName, request, triggerSource } = event;
  const { email, given_name, family_name } = request.userAttributes;

  // 確認後のみ処理（パスワードリセット確認などは除外）
  if (triggerSource !== "PostConfirmation_ConfirmSignUp") {
    return event;
  }

  try {
    // 1. DynamoDB にユーザーレコードを作成
    await docClient.send(
      new PutCommand({
        TableName: USERS_TABLE,
        Item: {
          PK: `USER#${userName}`,
          SK: `PROFILE`,
          userId: userName,
          email,
          givenName: given_name || "",
          familyName: family_name || "",
          createdAt: new Date().toISOString(),
          updatedAt: new Date().toISOString(),
        },
        ConditionExpression: "attribute_not_exists(PK)",
      })
    );

    // 2. デフォルトグループに追加
    await cognitoClient.send(
      new AdminAddUserToGroupCommand({
        UserPoolId: userPoolId,
        Username: userName,
        GroupName: "Member", // デフォルトで Member グループに追加
      })
    );

    console.log(`User ${email} created and added to Member group`);
  } catch (error) {
    console.error("Error in post confirmation:", error);
    // エラーでもサインアップは成功させる（ログで追跡）
  }

  return event;
};
```

### Pre Token Generation トリガー

```typescript
// infra/lambda/pre-token-generation/index.ts
import type {
  PreTokenGenerationTriggerEvent,
  PreTokenGenerationTriggerHandler,
} from "aws-lambda";
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import { DynamoDBDocumentClient, GetCommand } from "@aws-sdk/lib-dynamodb";

const dynamoClient = new DynamoDBClient({});
const docClient = DynamoDBDocumentClient.from(dynamoClient);

const USERS_TABLE = process.env.USERS_TABLE || "taskflow-users";

export const handler: PreTokenGenerationTriggerHandler = async (event) => {
  console.log("Pre Token Generation trigger:", JSON.stringify(event, null, 2));

  const { userName, request, response } = event;

  try {
    // DynamoDB からユーザーの追加情報を取得
    const result = await docClient.send(
      new GetCommand({
        TableName: USERS_TABLE,
        Key: {
          PK: `USER#${userName}`,
          SK: "PROFILE",
        },
      })
    );

    const userProfile = result.Item;

    // ID トークンにカスタムクレームを追加
    response.claimsOverrideDetails = {
      claimsToAddOrOverride: {
        // 組織 ID を追加
        "custom:organizationId": userProfile?.organizationId || "",
        // ユーザーのフルネームを追加
        "custom:fullName": `${userProfile?.familyName || ""} ${
          userProfile?.givenName || ""
        }`.trim(),
      },
      // 不要なクレームを抑制
      claimsToSuppress: [
        // 'phone_number', // 必要に応じて
      ],
    };

    // グループクレームをカスタマイズ（オプション）
    if (request.groupConfiguration?.groupsToOverride) {
      response.claimsOverrideDetails.groupOverrideDetails = {
        groupsToOverride: request.groupConfiguration.groupsToOverride,
      };
    }
  } catch (error) {
    console.error("Error in pre token generation:", error);
    // エラーでもトークン生成は続行
  }

  return event;
};
```

### CDK での Lambda Triggers 設定

```typescript
// infra/lib/cognito-stack.ts（Lambda Triggers を追加）
import * as cdk from "aws-cdk-lib";
import * as cognito from "aws-cdk-lib/aws-cognito";
import * as lambda from "aws-cdk-lib/aws-lambda";
import * as nodejs from "aws-cdk-lib/aws-lambda-nodejs";
import * as iam from "aws-cdk-lib/aws-iam";
import { Construct } from "constructs";
import * as path from "path";

interface CognitoStackProps extends cdk.StackProps {
  usersTableName: string;
}

export class CognitoStack extends cdk.Stack {
  public readonly userPool: cognito.UserPool;

  constructor(scope: Construct, id: string, props: CognitoStackProps) {
    super(scope, id, props);

    // Pre Sign-up Lambda
    const preSignUpFn = new nodejs.NodejsFunction(this, "PreSignUpFunction", {
      entry: path.join(__dirname, "../lambda/pre-signup/index.ts"),
      handler: "handler",
      runtime: lambda.Runtime.NODEJS_20_X,
      timeout: cdk.Duration.seconds(10),
    });

    // Post Confirmation Lambda
    const postConfirmationFn = new nodejs.NodejsFunction(
      this,
      "PostConfirmationFunction",
      {
        entry: path.join(__dirname, "../lambda/post-confirmation/index.ts"),
        handler: "handler",
        runtime: lambda.Runtime.NODEJS_20_X,
        timeout: cdk.Duration.seconds(10),
        environment: {
          USERS_TABLE: props.usersTableName,
        },
      }
    );

    // Pre Token Generation Lambda
    const preTokenGenerationFn = new nodejs.NodejsFunction(
      this,
      "PreTokenGenerationFunction",
      {
        entry: path.join(__dirname, "../lambda/pre-token-generation/index.ts"),
        handler: "handler",
        runtime: lambda.Runtime.NODEJS_20_X,
        timeout: cdk.Duration.seconds(10),
        environment: {
          USERS_TABLE: props.usersTableName,
        },
      }
    );

    // User Pool
    this.userPool = new cognito.UserPool(this, "TaskFlowUserPool", {
      userPoolName: "taskflow-user-pool",
      // ... 既存の設定 ...

      // Lambda Triggers
      lambdaTriggers: {
        preSignUp: preSignUpFn,
        postConfirmation: postConfirmationFn,
        preTokenGeneration: preTokenGenerationFn,
      },
    });

    // Post Confirmation Lambda に Cognito 権限を付与
    postConfirmationFn.addToRolePolicy(
      new iam.PolicyStatement({
        actions: ["cognito-idp:AdminAddUserToGroup"],
        resources: [this.userPool.userPoolArn],
      })
    );

    // DynamoDB 権限を付与
    const dynamoPolicy = new iam.PolicyStatement({
      actions: ["dynamodb:PutItem", "dynamodb:GetItem"],
      resources: [
        `arn:aws:dynamodb:${this.region}:${this.account}:table/${props.usersTableName}`,
      ],
    });
    postConfirmationFn.addToRolePolicy(dynamoPolicy);
    preTokenGenerationFn.addToRolePolicy(dynamoPolicy);
  }
}
```

## 実装タスク

### タスク 1: Custom Message トリガーの実装

メール検証コードのメッセージをカスタマイズするトリガーを実装してください。

<details>
<summary>回答</summary>

```typescript
// infra/lambda/custom-message/index.ts
import type {
  CustomMessageTriggerEvent,
  CustomMessageTriggerHandler,
} from "aws-lambda";

export const handler: CustomMessageTriggerHandler = async (event) => {
  console.log("Custom Message trigger:", JSON.stringify(event, null, 2));

  const { triggerSource, request, response } = event;
  const { codeParameter, userAttributes } = request;
  const userName =
    userAttributes.given_name ||
    userAttributes.email?.split("@")[0] ||
    "ユーザー";

  switch (triggerSource) {
    case "CustomMessage_SignUp":
      // サインアップ時の確認メール
      response.emailSubject = "【TaskFlow】メールアドレスの確認";
      response.emailMessage = `
        <html>
          <body style="font-family: sans-serif; padding: 20px;">
            <h1>TaskFlow へようこそ！</h1>
            <p>${userName} 様</p>
            <p>アカウント登録ありがとうございます。</p>
            <p>以下の確認コードを入力して、メールアドレスを確認してください：</p>
            <div style="background: #f5f5f5; padding: 20px; text-align: center; font-size: 24px; letter-spacing: 5px;">
              <strong>${codeParameter}</strong>
            </div>
            <p style="color: #666; font-size: 12px;">
              このコードは 24 時間有効です。
            </p>
          </body>
        </html>
      `;
      break;

    case "CustomMessage_ForgotPassword":
      // パスワードリセットメール
      response.emailSubject = "【TaskFlow】パスワードリセット";
      response.emailMessage = `
        <html>
          <body style="font-family: sans-serif; padding: 20px;">
            <h1>パスワードリセット</h1>
            <p>${userName} 様</p>
            <p>パスワードリセットのリクエストを受け付けました。</p>
            <p>以下の確認コードを入力して、新しいパスワードを設定してください：</p>
            <div style="background: #f5f5f5; padding: 20px; text-align: center; font-size: 24px; letter-spacing: 5px;">
              <strong>${codeParameter}</strong>
            </div>
            <p style="color: #666; font-size: 12px;">
              このリクエストに心当たりがない場合は、このメールを無視してください。
            </p>
          </body>
        </html>
      `;
      break;

    case "CustomMessage_ResendCode":
      // 確認コード再送信
      response.emailSubject = "【TaskFlow】確認コードの再送信";
      response.emailMessage = `
        <html>
          <body style="font-family: sans-serif; padding: 20px;">
            <h1>確認コード</h1>
            <p>新しい確認コードです：</p>
            <div style="background: #f5f5f5; padding: 20px; text-align: center; font-size: 24px; letter-spacing: 5px;">
              <strong>${codeParameter}</strong>
            </div>
          </body>
        </html>
      `;
      break;
  }

  return event;
};
```

</details>

## よくある間違い

### ❌ トリガーでエラーを投げてサインアップを中断

```typescript
// 悪い例: 重要でないエラーでサインアップを中断
export const handler = async (event) => {
  try {
    await sendWelcomeEmail(event.request.userAttributes.email);
  } catch (error) {
    throw error; // メール送信失敗でサインアップが失敗する
  }
  return event;
};
```

### ✅ 重要でないエラーはログに記録して続行

```typescript
// 良い例: エラーをログに記録して続行
export const handler = async (event) => {
  try {
    await sendWelcomeEmail(event.request.userAttributes.email);
  } catch (error) {
    console.error("Failed to send welcome email:", error);
    // サインアップは成功させる
  }
  return event;
};
```

## まとめ

この章では以下を学びました：

- Lambda Triggers の種類と実行タイミング
- Pre Sign-up でのサインアップカスタマイズ
- Post Confirmation での初期データ作成
- Pre Token Generation でのトークンカスタマイズ

## 確認クイズ

<details>
<summary>Q1: Pre Sign-up と Post Confirmation の違いは？</summary>

**A1:**

- Pre Sign-up: サインアップ処理の前に実行。ユーザー作成を拒否したり、自動確認を設定できる
- Post Confirmation: ユーザーがメール確認を完了した後に実行。DB へのレコード作成などに使用

Pre Sign-up でエラーを投げるとサインアップが失敗しますが、Post Confirmation でエラーを投げてもユーザーは既に作成されています。

</details>

<details>
<summary>Q2: Pre Token Generation で追加できるクレームの制限は？</summary>

**A2:**

- カスタムクレームは `custom:` プレフィックスを付ける必要がある
- 既存の標準クレーム（sub, email など）は上書きできない
- クレームの合計サイズには制限がある（トークンサイズ制限）
- Access Token のカスタマイズには追加料金がかかる場合がある
</details>

---

前の章: [08 - グループと権限管理](./08-groups-permissions.md)
次の章: [10 - Identity Pools と AWS リソース連携](./10-identity-pools.md)
