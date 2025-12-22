# 01 - Cognito の基礎と環境構築

## 学習の目的

- Amazon Cognito の基本概念（User Pools / Identity Pools）を理解する
- 認証フローの種類と選択基準を学ぶ
- AWS CDK を使用して Cognito リソースを構築する
- 開発環境をセットアップする

## 背景知識

### なぜ Cognito を使うのか？

認証システムを自前で構築する場合、以下の課題に対処する必要があります：

- パスワードの安全な保存（ハッシュ化、ソルト）
- セッション管理とトークン発行
- MFA の実装
- ソーシャルログインの統合
- セキュリティ脅威への対応（ブルートフォース攻撃、認証情報の漏洩）

Amazon Cognito はこれらを AWS マネージドサービスとして提供し、開発者は認証ロジックではなくアプリケーションの価値に集中できます。

### User Pools vs Identity Pools

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Amazon Cognito                                   │
│                                                                          │
│  ┌─────────────────────────────┐    ┌─────────────────────────────────┐│
│  │       User Pools            │    │        Identity Pools           ││
│  │                             │    │                                 ││
│  │  "誰であるか" を管理        │    │  "何ができるか" を管理          ││
│  │                             │    │                                 ││
│  │  • ユーザーディレクトリ     │    │  • AWS 認証情報の発行           ││
│  │  • 認証（サインイン）       │    │  • IAM ロールとの連携           ││
│  │  • JWT トークン発行         │    │  • 一時的なアクセス権限         ││
│  │  • MFA、パスキー            │    │                                 ││
│  │  • ソーシャルログイン       │    │                                 ││
│  │                             │    │                                 ││
│  │  出力: ID Token,            │───▶│  入力: ID Token                 ││
│  │       Access Token,         │    │  出力: AWS Credentials          ││
│  │       Refresh Token         │    │        (AccessKeyId,            ││
│  │                             │    │         SecretAccessKey,        ││
│  │                             │    │         SessionToken)           ││
│  └─────────────────────────────┘    └─────────────────────────────────┘│
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

| 機能         | User Pools               | Identity Pools                 |
| ------------ | ------------------------ | ------------------------------ |
| 主な目的     | ユーザー認証             | AWS リソースへのアクセス認可   |
| 出力         | JWT トークン             | 一時的な AWS 認証情報          |
| ユースケース | API 認証、アプリログイン | S3 直接アクセス、DynamoDB 操作 |
| 単独使用     | 可能                     | 可能（他の IdP と組み合わせ）  |

### 認証フローの種類

Cognito は複数の認証フローをサポートしています：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        認証フローの選択                                  │
│                                                                          │
│  ┌─────────────────────┐                                                │
│  │ パスワードベース    │                                                │
│  │                     │                                                │
│  │ USER_PASSWORD_AUTH  │──▶ シンプル、パスワードを直接送信              │
│  │ USER_SRP_AUTH       │──▶ セキュア、パスワードを送信しない（推奨）    │
│  └─────────────────────┘                                                │
│                                                                          │
│  ┌─────────────────────┐                                                │
│  │ パスワードレス      │                                                │
│  │                     │                                                │
│  │ EMAIL_OTP           │──▶ メールで OTP を送信                         │
│  │ SMS_OTP             │──▶ SMS で OTP を送信                           │
│  │ WEB_AUTHN           │──▶ パスキー（生体認証、セキュリティキー）      │
│  └─────────────────────┘                                                │
│                                                                          │
│  ┌─────────────────────┐                                                │
│  │ フェデレーション    │                                                │
│  │                     │                                                │
│  │ OIDC / SAML         │──▶ 外部 IdP（Google、企業 IdP など）           │
│  └─────────────────────┘                                                │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

## 概念の説明

### JWT トークンの構造

Cognito が発行する JWT トークンは 3 種類あります：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          JWT トークン                                    │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ ID Token                                                         │   │
│  │ • ユーザーの属性情報（email, name, custom attributes）          │   │
│  │ • 認証されたユーザーの身元を証明                                 │   │
│  │ • フロントエンドでユーザー情報を表示する際に使用                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Access Token                                                     │   │
│  │ • API へのアクセス権限を証明                                     │   │
│  │ • スコープ情報を含む                                             │   │
│  │ • バックエンド API の認証に使用                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Refresh Token                                                    │   │
│  │ • 新しい ID/Access Token を取得するために使用                    │   │
│  │ • 長い有効期限（デフォルト 30 日）                               │   │
│  │ • セキュアに保存する必要がある                                   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### App Client の設定

App Client は、アプリケーションが Cognito と通信するための設定です：

```typescript
// App Client の主要な設定項目
{
  // 認証フロー
  authFlows: {
    userPassword: true,      // USER_PASSWORD_AUTH
    userSrp: true,           // USER_SRP_AUTH（推奨）
    custom: true,            // CUSTOM_AUTH（Lambda トリガー）
  },

  // OAuth 設定
  oAuth: {
    flows: {
      authorizationCodeGrant: true,  // 推奨
      implicitCodeGrant: false,      // 非推奨
    },
    scopes: ['openid', 'email', 'profile'],
    callbackUrls: ['http://localhost:4200/callback'],
    logoutUrls: ['http://localhost:4200/logout'],
  },

  // トークン有効期限
  accessTokenValidity: Duration.hours(1),
  idTokenValidity: Duration.hours(1),
  refreshTokenValidity: Duration.days(30),
}
```

## 環境構築

### 前提条件

以下がインストールされていることを確認してください：

- Node.js 20+
- AWS CLI（設定済み）
- AWS CDK CLI

```bash
# バージョン確認
node --version  # v20.x.x
aws --version   # aws-cli/2.x.x
cdk --version   # 2.x.x
```

### プロジェクトの初期化

```bash
# プロジェクトディレクトリの作成
mkdir taskflow && cd taskflow

# monorepo の初期化
npm init -y

# 必要なディレクトリの作成
mkdir -p apps/web apps/api infra packages/shared docs/guide
```

### CDK プロジェクトのセットアップ

```bash
cd infra
npx cdk init app --language typescript
```

### 依存関係のインストール

```bash
# infra ディレクトリで
npm install aws-cdk-lib constructs
npm install -D typescript @types/node
```

## 実装タスク

### タスク 1: Cognito User Pool の CDK 定義

以下の要件を満たす User Pool を CDK で定義してください：

**要件:**

- メールアドレスでサインアップ
- メール検証を必須にする
- パスワードポリシー: 最小 8 文字、大文字・小文字・数字・記号を含む
- カスタム属性: `organizationId` (文字列)

<details>
<summary>ヒント</summary>

`aws-cdk-lib/aws-cognito` モジュールの `UserPool` クラスを使用します。
`signInAliases` でサインイン方法を、`standardAttributes` と `customAttributes` で属性を定義します。

</details>

<details>
<summary>回答</summary>

```typescript
// infra/lib/cognito-stack.ts
import * as cdk from "aws-cdk-lib";
import * as cognito from "aws-cdk-lib/aws-cognito";
import { Construct } from "constructs";

export class CognitoStack extends cdk.Stack {
  public readonly userPool: cognito.UserPool;
  public readonly userPoolClient: cognito.UserPoolClient;

  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // User Pool の作成
    this.userPool = new cognito.UserPool(this, "TaskFlowUserPool", {
      userPoolName: "taskflow-user-pool",

      // サインイン設定
      signInAliases: {
        email: true,
        username: false,
      },

      // 自己サインアップを許可
      selfSignUpEnabled: true,

      // メール検証を必須に
      autoVerify: {
        email: true,
      },

      // 標準属性
      standardAttributes: {
        email: {
          required: true,
          mutable: true,
        },
        givenName: {
          required: false,
          mutable: true,
        },
        familyName: {
          required: false,
          mutable: true,
        },
      },

      // カスタム属性
      customAttributes: {
        organizationId: new cognito.StringAttribute({
          mutable: true,
        }),
      },

      // パスワードポリシー
      passwordPolicy: {
        minLength: 8,
        requireLowercase: true,
        requireUppercase: true,
        requireDigits: true,
        requireSymbols: true,
        tempPasswordValidity: cdk.Duration.days(7),
      },

      // アカウント復旧設定
      accountRecovery: cognito.AccountRecovery.EMAIL_ONLY,

      // 削除保護（本番環境では RETAIN に）
      removalPolicy: cdk.RemovalPolicy.DESTROY,
    });

    // App Client の作成
    this.userPoolClient = this.userPool.addClient("TaskFlowWebClient", {
      userPoolClientName: "taskflow-web-client",

      // 認証フロー
      authFlows: {
        userPassword: true,
        userSrp: true,
        custom: true,
      },

      // トークン有効期限
      accessTokenValidity: cdk.Duration.hours(1),
      idTokenValidity: cdk.Duration.hours(1),
      refreshTokenValidity: cdk.Duration.days(30),

      // クライアントシークレットを生成しない（SPA 用）
      generateSecret: false,

      // 認証エラーの詳細を防ぐ
      preventUserExistenceErrors: true,
    });

    // 出力
    new cdk.CfnOutput(this, "UserPoolId", {
      value: this.userPool.userPoolId,
      description: "Cognito User Pool ID",
    });

    new cdk.CfnOutput(this, "UserPoolClientId", {
      value: this.userPoolClient.userPoolClientId,
      description: "Cognito User Pool Client ID",
    });

    new cdk.CfnOutput(this, "UserPoolRegion", {
      value: this.region,
      description: "AWS Region",
    });
  }
}
```

</details>

### タスク 2: CDK アプリケーションのエントリポイント

CDK アプリケーションのエントリポイントを作成してください。

<details>
<summary>回答</summary>

```typescript
// infra/bin/infra.ts
#!/usr/bin/env node
import 'source-map-support/register';
import * as cdk from 'aws-cdk-lib';
import { CognitoStack } from '../lib/cognito-stack';

const app = new cdk.App();

const env = {
  account: process.env.CDK_DEFAULT_ACCOUNT,
  region: process.env.CDK_DEFAULT_REGION || 'ap-northeast-1',
};

new CognitoStack(app, 'TaskFlowCognitoStack', {
  env,
  description: 'TaskFlow Cognito User Pool and related resources',
});
```

</details>

### タスク 3: デプロイと動作確認

```bash
# infra ディレクトリで
cd infra

# CDK Bootstrap（初回のみ）
npx cdk bootstrap

# 差分確認
npx cdk diff

# デプロイ
npx cdk deploy

# 出力された UserPoolId と UserPoolClientId をメモ
```

デプロイ後、AWS コンソールで以下を確認してください：

1. Cognito > User pools に `taskflow-user-pool` が作成されている
2. App integration タブで `taskflow-web-client` が作成されている
3. Sign-in experience タブでパスワードポリシーが設定されている

## よくある間違い

### ❌ クライアントシークレットを SPA で使用

```typescript
// 悪い例: SPA でクライアントシークレットを使用
this.userPool.addClient("WebClient", {
  generateSecret: true, // SPA ではシークレットを安全に保存できない
});
```

### ✅ SPA ではシークレットなしで設定

```typescript
// 良い例: SPA ではシークレットを生成しない
this.userPool.addClient("WebClient", {
  generateSecret: false,
  preventUserExistenceErrors: true, // セキュリティ強化
});
```

### ❌ 本番環境で RemovalPolicy.DESTROY

```typescript
// 悪い例: 本番環境でユーザーデータが削除される可能性
removalPolicy: cdk.RemovalPolicy.DESTROY,
```

### ✅ 本番環境では RETAIN を使用

```typescript
// 良い例: スタック削除時もユーザーデータを保持
removalPolicy: cdk.RemovalPolicy.RETAIN,
```

## まとめ

この章では以下を学びました：

- Cognito の 2 つの主要コンポーネント（User Pools / Identity Pools）の違い
- JWT トークンの種類と用途
- CDK を使用した User Pool の構築
- App Client の設定とセキュリティ考慮事項

## 確認クイズ

<details>
<summary>Q1: User Pools と Identity Pools の主な違いは何ですか？</summary>

**A1:**

- User Pools: ユーザー認証を担当し、JWT トークンを発行する
- Identity Pools: AWS リソースへのアクセス認可を担当し、一時的な AWS 認証情報を発行する

User Pools は「誰であるか」を、Identity Pools は「何ができるか」を管理します。

</details>

<details>
<summary>Q2: SPA で generateSecret: true を設定すべきでない理由は？</summary>

**A2:**
SPA（Single Page Application）はブラウザ上で動作するため、クライアントシークレットをソースコードに含めると、誰でも閲覧可能になります。シークレットが漏洩すると、攻撃者がアプリケーションになりすまして API を呼び出せてしまいます。

SPA では PKCE（Proof Key for Code Exchange）を使用した認証フローを使用し、シークレットなしで安全に認証を行います。

</details>

<details>
<summary>Q3: preventUserExistenceErrors: true の効果は？</summary>

**A3:**
この設定を有効にすると、存在しないユーザーでログインを試みた場合でも、「ユーザーが存在しない」というエラーではなく、「認証に失敗しました」という一般的なエラーを返します。

これにより、攻撃者がユーザー名の存在を確認する「ユーザー列挙攻撃」を防ぐことができます。

</details>

---

前の章: [00 - 概要](./00-overview.md)
次の章: [02 - 基本認証フローの実装](./02-basic-auth-flow.md)
