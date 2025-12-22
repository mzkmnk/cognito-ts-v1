# 12 - 本番環境への展開

## 学習の目的

- 本番環境向けのセキュリティベストプラクティスを学ぶ
- 環境ごとの設定管理方法を理解する
- 監視とアラートの設定方法を学ぶ
- 運用時の注意点を把握する

## 背景知識

### 本番環境チェックリスト

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     本番環境デプロイチェックリスト                       │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ セキュリティ                                                     │   │
│  │ □ Advanced Security を ENFORCED モードに設定                     │   │
│  │ □ MFA を有効化（OPTIONAL または ON）                             │   │
│  │ □ パスワードポリシーを強化                                       │   │
│  │ □ トークン有効期限を適切に設定                                   │   │
│  │ □ HTTPS のみを許可                                               │   │
│  │ □ CORS を本番ドメインのみに制限                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 可用性                                                           │   │
│  │ □ Lambda のプロビジョニング済み同時実行を設定                    │   │
│  │ □ DynamoDB のキャパシティを適切に設定                            │   │
│  │ □ CloudFront でキャッシュを最適化                                │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 監視                                                             │   │
│  │ □ CloudWatch アラームを設定                                      │   │
│  │ □ ログの保持期間を設定                                           │   │
│  │ □ セキュリティイベントの通知を設定                               │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ バックアップ                                                     │   │
│  │ □ DynamoDB のポイントインタイムリカバリを有効化                  │   │
│  │ □ S3 のバージョニングを有効化                                    │   │
│  │ □ User Pool の削除保護を有効化                                   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 環境分離

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     環境分離の構成                                       │
│                                                                          │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────┐ │
│  │ Development     │  │ Staging         │  │ Production              │ │
│  │                 │  │                 │  │                         │ │
│  │ • ローカル開発  │  │ • 本番同等環境  │  │ • 本番環境              │ │
│  │ • 自由にテスト  │  │ • E2E テスト    │  │ • 実ユーザー            │ │
│  │ • 削除可能      │  │ • 性能テスト    │  │ • 削除保護あり          │ │
│  │                 │  │                 │  │                         │ │
│  │ User Pool:      │  │ User Pool:      │  │ User Pool:              │ │
│  │ taskflow-dev    │  │ taskflow-stg    │  │ taskflow-prod           │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────────────┘ │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

## 概念の説明

### セキュリティベストプラクティス

1. **最小権限の原則**: 必要最小限の権限のみを付与
2. **多層防御**: 複数のセキュリティレイヤーを設置
3. **監査ログ**: 全ての認証イベントを記録
4. **定期的なレビュー**: セキュリティ設定を定期的に見直し

### トークン有効期限の設計

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     トークン有効期限の推奨設定                           │
│                                                                          │
│  トークン種類        推奨有効期限      理由                              │
│  ─────────────────────────────────────────────────────────────────────  │
│                                                                          │
│  Access Token       15分〜1時間       短いほど安全、頻繁なリフレッシュ  │
│                                                                          │
│  ID Token           15分〜1時間       Access Token と同じ                │
│                                                                          │
│  Refresh Token      7日〜30日         長すぎると漏洩時のリスク増大      │
│                                                                          │
│  ※ セキュリティ要件に応じて調整                                         │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

## コードサンプル

### 本番環境用 CDK 設定

```typescript
// infra/lib/production-cognito-stack.ts
import * as cdk from "aws-cdk-lib";
import * as cognito from "aws-cdk-lib/aws-cognito";
import { Construct } from "constructs";

interface ProductionCognitoStackProps extends cdk.StackProps {
  domainName: string;
  sesEmailArn: string;
}

export class ProductionCognitoStack extends cdk.Stack {
  public readonly userPool: cognito.UserPool;

  constructor(
    scope: Construct,
    id: string,
    props: ProductionCognitoStackProps
  ) {
    super(scope, id, props);

    this.userPool = new cognito.UserPool(this, "ProductionUserPool", {
      userPoolName: "taskflow-prod",

      // サインイン設定
      signInAliases: { email: true },
      selfSignUpEnabled: true,
      autoVerify: { email: true },

      // 強力なパスワードポリシー
      passwordPolicy: {
        minLength: 12, // 本番では12文字以上推奨
        requireLowercase: true,
        requireUppercase: true,
        requireDigits: true,
        requireSymbols: true,
        tempPasswordValidity: cdk.Duration.days(3), // 一時パスワードは3日で期限切れ
      },

      // MFA 設定
      mfa: cognito.Mfa.OPTIONAL, // または REQUIRED
      mfaSecondFactor: {
        sms: false, // SMS は非推奨（コストとセキュリティ）
        otp: true,
      },

      // Advanced Security
      advancedSecurityMode: cognito.AdvancedSecurityMode.ENFORCED,

      // アカウント復旧
      accountRecovery: cognito.AccountRecovery.EMAIL_ONLY,

      // 削除保護（本番環境では必須）
      removalPolicy: cdk.RemovalPolicy.RETAIN,
      deletionProtection: true,

      // メール設定（SES を使用）
      email: cognito.UserPoolEmail.withSES({
        sesRegion: this.region,
        fromEmail: `noreply@${props.domainName}`,
        fromName: "TaskFlow",
        sesVerifiedDomain: props.domainName,
      }),

      // ユーザー属性
      standardAttributes: {
        email: { required: true, mutable: true },
        givenName: { required: false, mutable: true },
        familyName: { required: false, mutable: true },
      },
      customAttributes: {
        organizationId: new cognito.StringAttribute({ mutable: true }),
      },

      // デバイス追跡
      deviceTracking: {
        challengeRequiredOnNewDevice: true,
        deviceOnlyRememberedOnUserPrompt: true,
      },
    });

    // App Client（本番設定）
    const userPoolClient = this.userPool.addClient("ProductionWebClient", {
      userPoolClientName: "taskflow-prod-web",

      authFlows: {
        userPassword: false, // SRP のみ許可（より安全）
        userSrp: true,
        custom: true,
      },

      // 短いトークン有効期限
      accessTokenValidity: cdk.Duration.minutes(30),
      idTokenValidity: cdk.Duration.minutes(30),
      refreshTokenValidity: cdk.Duration.days(7),

      // セキュリティ設定
      generateSecret: false,
      preventUserExistenceErrors: true,
      enableTokenRevocation: true,

      // OAuth 設定
      oAuth: {
        flows: {
          authorizationCodeGrant: true,
          implicitCodeGrant: false,
        },
        scopes: [
          cognito.OAuthScope.OPENID,
          cognito.OAuthScope.EMAIL,
          cognito.OAuthScope.PROFILE,
        ],
        callbackUrls: [`https://${props.domainName}/auth/callback`],
        logoutUrls: [`https://${props.domainName}/`],
      },
    });

    // WebAuthn 設定
    const cfnUserPool = this.userPool.node.defaultChild as cognito.CfnUserPool;
    cfnUserPool.addPropertyOverride("WebAuthnConfiguration", {
      RelyingPartyId: props.domainName,
      UserVerification: "required", // 本番では required 推奨
    });
  }
}
```

### 環境変数の管理

```typescript
// infra/bin/infra.ts
#!/usr/bin/env node
import 'source-map-support/register';
import * as cdk from 'aws-cdk-lib';
import { ProductionCognitoStack } from '../lib/production-cognito-stack';

const app = new cdk.App();

// 環境の取得
const environment = app.node.tryGetContext('environment') || 'dev';

// 環境ごとの設定
const envConfig: Record<string, {
  domainName: string;
  account: string;
  region: string;
}> = {
  dev: {
    domainName: 'dev.taskflow.example.com',
    account: '111111111111',
    region: 'ap-northeast-1',
  },
  staging: {
    domainName: 'staging.taskflow.example.com',
    account: '222222222222',
    region: 'ap-northeast-1',
  },
  prod: {
    domainName: 'taskflow.example.com',
    account: '333333333333',
    region: 'ap-northeast-1',
  },
};

const config = envConfig[environment];

new ProductionCognitoStack(app, `TaskFlow-${environment}-Cognito`, {
  env: {
    account: config.account,
    region: config.region,
  },
  domainName: config.domainName,
  sesEmailArn: `arn:aws:ses:${config.region}:${config.account}:identity/${config.domainName}`,
});
```

### デプロイスクリプト

```bash
#!/bin/bash
# deploy.sh

set -e

ENVIRONMENT=${1:-dev}

echo "Deploying to $ENVIRONMENT environment..."

# 環境変数の検証
if [[ ! "$ENVIRONMENT" =~ ^(dev|staging|prod)$ ]]; then
  echo "Invalid environment: $ENVIRONMENT"
  echo "Usage: ./deploy.sh [dev|staging|prod]"
  exit 1
fi

# 本番環境の場合は確認
if [ "$ENVIRONMENT" = "prod" ]; then
  read -p "Are you sure you want to deploy to PRODUCTION? (yes/no): " confirm
  if [ "$confirm" != "yes" ]; then
    echo "Deployment cancelled."
    exit 0
  fi
fi

# CDK デプロイ
cd infra
npx cdk deploy --all --context environment=$ENVIRONMENT --require-approval never

echo "Deployment to $ENVIRONMENT completed!"
```

### 監視とアラートの設定

```typescript
// infra/lib/monitoring-stack.ts
import * as cdk from "aws-cdk-lib";
import * as cloudwatch from "aws-cdk-lib/aws-cloudwatch";
import * as sns from "aws-cdk-lib/aws-sns";
import * as snsSubscriptions from "aws-cdk-lib/aws-sns-subscriptions";
import * as cloudwatchActions from "aws-cdk-lib/aws-cloudwatch-actions";
import { Construct } from "constructs";

interface MonitoringStackProps extends cdk.StackProps {
  userPoolId: string;
  alertEmail: string;
}

export class MonitoringStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props: MonitoringStackProps) {
    super(scope, id, props);

    // アラート通知用 SNS トピック
    const alertTopic = new sns.Topic(this, "AlertTopic", {
      topicName: "taskflow-alerts",
    });

    // メール通知
    alertTopic.addSubscription(
      new snsSubscriptions.EmailSubscription(props.alertEmail)
    );

    // サインイン失敗率のアラーム
    const signInFailureAlarm = new cloudwatch.Alarm(
      this,
      "SignInFailureAlarm",
      {
        alarmName: "TaskFlow-SignInFailureRate",
        alarmDescription: "サインイン失敗率が高くなっています",
        metric: new cloudwatch.MathExpression({
          expression: "(failures / (successes + failures)) * 100",
          usingMetrics: {
            failures: new cloudwatch.Metric({
              namespace: "AWS/Cognito",
              metricName: "SignInFailures",
              dimensionsMap: { UserPoolId: props.userPoolId },
              statistic: "Sum",
              period: cdk.Duration.minutes(5),
            }),
            successes: new cloudwatch.Metric({
              namespace: "AWS/Cognito",
              metricName: "SignInSuccesses",
              dimensionsMap: { UserPoolId: props.userPoolId },
              statistic: "Sum",
              period: cdk.Duration.minutes(5),
            }),
          },
        }),
        threshold: 50, // 50% 以上で警告
        evaluationPeriods: 3,
        comparisonOperator:
          cloudwatch.ComparisonOperator.GREATER_THAN_THRESHOLD,
      }
    );

    signInFailureAlarm.addAlarmAction(
      new cloudwatchActions.SnsAction(alertTopic)
    );

    // Lambda エラーのアラーム
    const lambdaErrorAlarm = new cloudwatch.Alarm(this, "LambdaErrorAlarm", {
      alarmName: "TaskFlow-LambdaErrors",
      alarmDescription: "Lambda 関数でエラーが発生しています",
      metric: new cloudwatch.Metric({
        namespace: "AWS/Lambda",
        metricName: "Errors",
        dimensionsMap: { FunctionName: "TaskFlowApi" },
        statistic: "Sum",
        period: cdk.Duration.minutes(5),
      }),
      threshold: 10,
      evaluationPeriods: 2,
      comparisonOperator: cloudwatch.ComparisonOperator.GREATER_THAN_THRESHOLD,
    });

    lambdaErrorAlarm.addAlarmAction(
      new cloudwatchActions.SnsAction(alertTopic)
    );

    // ダッシュボード
    new cloudwatch.Dashboard(this, "OperationsDashboard", {
      dashboardName: "TaskFlow-Operations",
      widgets: [
        [
          new cloudwatch.GraphWidget({
            title: "認証メトリクス",
            left: [
              new cloudwatch.Metric({
                namespace: "AWS/Cognito",
                metricName: "SignInSuccesses",
                dimensionsMap: { UserPoolId: props.userPoolId },
                statistic: "Sum",
                period: cdk.Duration.hours(1),
                label: "サインイン成功",
              }),
              new cloudwatch.Metric({
                namespace: "AWS/Cognito",
                metricName: "SignInFailures",
                dimensionsMap: { UserPoolId: props.userPoolId },
                statistic: "Sum",
                period: cdk.Duration.hours(1),
                label: "サインイン失敗",
              }),
            ],
            width: 12,
          }),
          new cloudwatch.GraphWidget({
            title: "API レイテンシ",
            left: [
              new cloudwatch.Metric({
                namespace: "AWS/ApiGateway",
                metricName: "Latency",
                statistic: "Average",
                period: cdk.Duration.minutes(5),
              }),
            ],
            width: 12,
          }),
        ],
      ],
    });
  }
}
```

## 運用時の注意点

### ユーザー管理

```typescript
// 管理者によるユーザー操作の例
import {
  CognitoIdentityProviderClient,
  AdminDisableUserCommand,
  AdminEnableUserCommand,
  AdminResetUserPasswordCommand,
  AdminSetUserPasswordCommand,
} from "@aws-sdk/client-cognito-identity-provider";

const client = new CognitoIdentityProviderClient({});

// ユーザーを無効化
async function disableUser(userPoolId: string, username: string) {
  await client.send(
    new AdminDisableUserCommand({
      UserPoolId: userPoolId,
      Username: username,
    })
  );
}

// パスワードをリセット（一時パスワードを設定）
async function resetUserPassword(userPoolId: string, username: string) {
  await client.send(
    new AdminResetUserPasswordCommand({
      UserPoolId: userPoolId,
      Username: username,
    })
  );
}
```

### インシデント対応

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     インシデント対応フロー                               │
│                                                                          │
│  1. 検出                                                                 │
│     • CloudWatch アラームで検出                                         │
│     • ユーザーからの報告                                                │
│                                                                          │
│  2. 初期対応                                                             │
│     • 影響範囲の特定                                                    │
│     • 必要に応じてユーザーを無効化                                      │
│     • 関係者への通知                                                    │
│                                                                          │
│  3. 調査                                                                 │
│     • CloudWatch Logs でログを確認                                      │
│     • Cognito のユーザーアクティビティを確認                            │
│                                                                          │
│  4. 復旧                                                                 │
│     • 原因の修正                                                        │
│     • 影響を受けたユーザーへの対応                                      │
│                                                                          │
│  5. 事後対応                                                             │
│     • インシデントレポートの作成                                        │
│     • 再発防止策の実施                                                  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

## まとめ

この章では以下を学びました：

- 本番環境向けのセキュリティ設定
- 環境ごとの設定管理
- 監視とアラートの設定
- 運用時の注意点とインシデント対応

## 確認クイズ

<details>
<summary>Q1: 本番環境で deletionProtection を有効にする理由は？</summary>

**A1:**
誤ってスタックを削除した場合でも、User Pool が削除されないようにするためです。User Pool が削除されると、全ユーザーのデータが失われ、復旧が困難になります。

本番環境では必ず `deletionProtection: true` と `removalPolicy: cdk.RemovalPolicy.RETAIN` を設定してください。

</details>

<details>
<summary>Q2: トークン有効期限を短くするメリットとデメリットは？</summary>

**A2:**
メリット:

- トークンが漏洩した場合の影響を最小限に抑えられる
- セッションハイジャックのリスクを軽減

デメリット:

- 頻繁なトークンリフレッシュが必要
- ネットワーク遅延やエラー時にユーザー体験が低下する可能性

バランスを取って、Access Token は 15 分〜1 時間、Refresh Token は 7 日〜30 日が推奨されます。

</details>

## 学習ガイド完了

おめでとうございます！このガイドを通じて、Amazon Cognito の実務レベルの機能を網羅的に学習しました。

### 学習した内容の振り返り

1. **基礎**: User Pools と Identity Pools の概念
2. **認証**: サインアップ、サインイン、パスワードリセット
3. **UI**: Angular での認証画面実装
4. **API**: Hono での JWT 検証
5. **MFA**: TOTP による多要素認証
6. **パスキー**: WebAuthn によるパスワードレス認証
7. **ソーシャル**: Google ログインの統合
8. **権限**: グループベースのアクセス制御
9. **カスタマイズ**: Lambda Triggers
10. **AWS 連携**: Identity Pools と S3
11. **セキュリティ**: Advanced Security
12. **運用**: 本番環境への展開

### 次のステップ

- 実際にアプリケーションを構築して理解を深める
- AWS 公式ドキュメントで最新機能を確認する
- セキュリティベストプラクティスを定期的にレビューする

---

前の章: [11 - Advanced Security](./11-advanced-security.md)
概要に戻る: [00 - 概要](./00-overview.md)
