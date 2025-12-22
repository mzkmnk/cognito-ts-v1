# 11 - Advanced Security

## 学習の目的

- Cognito Advanced Security の機能を理解する
- 適応型認証（Adaptive Authentication）を設定する
- 侵害された認証情報の検出を有効化する
- セキュリティイベントの監視方法を学ぶ

## 背景知識

### Advanced Security とは

Cognito Advanced Security は、機械学習を使用してリスクベースの認証を提供する機能です：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     Advanced Security 機能                               │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 適応型認証（Adaptive Authentication）                            │   │
│  │                                                                   │   │
│  │ • リスクスコアに基づいて追加認証を要求                           │   │
│  │ • 新しいデバイス、異常な場所からのアクセスを検出                 │   │
│  │ • 自動的に MFA を要求またはブロック                              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 侵害された認証情報の検出                                         │   │
│  │                                                                   │   │
│  │ • 漏洩したパスワードデータベースとの照合                         │   │
│  │ • サインアップ時とパスワード変更時にチェック                     │   │
│  │ • 侵害が検出された場合はブロックまたは警告                       │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ ユーザーアクティビティログ                                       │   │
│  │                                                                   │   │
│  │ • 認証イベントの詳細なログ                                       │   │
│  │ • リスク評価の履歴                                               │   │
│  │ • CloudWatch との統合                                            │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### リスクレベルと対応

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     リスクレベルと対応アクション                         │
│                                                                          │
│  リスクレベル        対応オプション                                      │
│  ─────────────────────────────────────────────────────────────────────  │
│                                                                          │
│  Low（低）          • 許可（追加認証なし）                              │
│                                                                          │
│  Medium（中）       • MFA を要求                                        │
│                     • 許可（ログのみ）                                  │
│                                                                          │
│  High（高）         • ブロック                                          │
│                     • MFA を要求                                        │
│                     • 許可（ログのみ）                                  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### リスク評価の要素

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     リスク評価に使用される要素                           │
│                                                                          │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────┐ │
│  │ デバイス情報    │  │ 位置情報        │  │ 行動パターン            │ │
│  │                 │  │                 │  │                         │ │
│  │ • ブラウザ      │  │ • IP アドレス   │  │ • ログイン時間          │ │
│  │ • OS            │  │ • 地理的位置    │  │ • 操作パターン          │ │
│  │ • デバイス ID   │  │ • ISP           │  │ • 過去の履歴            │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────────────┘ │
│                                                                          │
│  これらの要素を機械学習で分析し、リスクスコアを算出                      │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

## 概念の説明

### 適応型認証のフロー

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     適応型認証フロー                                     │
│                                                                          │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────────────────┐ │
│  │ SignIn  │───▶│ リスク  │───▶│ 判定    │───▶│ アクション          │ │
│  │ Request │    │ 評価    │    │         │    │                     │ │
│  └─────────┘    └─────────┘    └─────────┘    └─────────────────────┘ │
│                      │              │                   │               │
│                      ▼              ▼                   ▼               │
│               デバイス、IP、    Low/Medium/High    許可/MFA/ブロック   │
│               行動を分析                                                │
│                                                                          │
│  例: 新しいデバイス + 異なる国からのアクセス = High リスク → MFA 要求  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

## コードサンプル

### CDK での Advanced Security 設定

```typescript
// infra/lib/cognito-stack.ts（Advanced Security を追加）
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

      // Advanced Security を有効化
      advancedSecurityMode: cognito.AdvancedSecurityMode.ENFORCED,
    });

    // L1 Construct で詳細設定
    const cfnUserPool = this.userPool.node.defaultChild as cognito.CfnUserPool;

    // ユーザープールのリスク設定
    cfnUserPool.userPoolAddOns = {
      advancedSecurityMode: "ENFORCED",
    };

    // アカウント乗っ取り防止設定
    cfnUserPool.addPropertyOverride(
      "UserPoolAddOns.AdvancedSecurityAdditionalFlows",
      {
        CustomAuthMode: "AUDIT", // カスタム認証のリスク評価
      }
    );
  }
}
```

### リスク設定の詳細（CloudFormation）

```typescript
// infra/lib/cognito-stack.ts（リスク設定を追加）

// User Pool Risk Configuration
new cognito.CfnUserPoolRiskConfigurationAttachment(this, "RiskConfiguration", {
  userPoolId: this.userPool.userPoolId,
  clientId: "ALL", // 全クライアントに適用

  // 侵害された認証情報の検出
  compromisedCredentialsRiskConfiguration: {
    actions: {
      eventAction: "BLOCK", // 侵害された認証情報をブロック
    },
    eventFilter: ["SIGN_IN", "SIGN_UP", "PASSWORD_CHANGE"],
  },

  // アカウント乗っ取り防止
  accountTakeoverRiskConfiguration: {
    actions: {
      // 低リスク: 許可
      lowAction: {
        eventAction: "NO_ACTION",
        notify: false,
      },
      // 中リスク: MFA を要求
      mediumAction: {
        eventAction: "MFA_IF_CONFIGURED",
        notify: true,
      },
      // 高リスク: ブロック
      highAction: {
        eventAction: "BLOCK",
        notify: true,
      },
    },
    notifyConfiguration: {
      sourceArn: `arn:aws:ses:${this.region}:${this.account}:identity/noreply@taskflow.example.com`,
      from: "TaskFlow Security <noreply@taskflow.example.com>",
      replyTo: "support@taskflow.example.com",
      blockEmail: {
        subject: "【TaskFlow】アカウントへの不審なアクセスをブロックしました",
        htmlBody: `
            <p>お客様のアカウントへの不審なアクセスを検出し、ブロックしました。</p>
            <p>心当たりがない場合は、パスワードを変更することをお勧めします。</p>
          `,
        textBody:
          "お客様のアカウントへの不審なアクセスを検出し、ブロックしました。",
      },
      mfaEmail: {
        subject: "【TaskFlow】追加の認証が必要です",
        htmlBody: `
            <p>通常と異なる環境からのアクセスを検出しました。</p>
            <p>セキュリティのため、追加の認証が必要です。</p>
          `,
        textBody: "通常と異なる環境からのアクセスを検出しました。",
      },
      noActionEmail: {
        subject: "【TaskFlow】新しいデバイスからのログイン",
        htmlBody: `
            <p>新しいデバイスからのログインがありました。</p>
            <p>心当たりがない場合は、すぐにパスワードを変更してください。</p>
          `,
        textBody: "新しいデバイスからのログインがありました。",
      },
    },
  },
});
```

### フロントエンドでのリスク情報の活用

```typescript
// apps/web/src/app/auth/services/auth.service.ts に追加

/**
 * サインイン結果からリスク情報を取得
 */
handleSignInResult(result: SignInOutput): void {
  // MFA が要求された場合（リスクベース）
  if (result.nextStep?.signInStep === 'CONFIRM_SIGN_IN_WITH_TOTP_CODE') {
    // 適応型認証により MFA が要求された
    console.log('Additional authentication required due to risk assessment');
  }
}
```

### CloudWatch でのセキュリティイベント監視

```typescript
// infra/lib/monitoring-stack.ts
import * as cdk from "aws-cdk-lib";
import * as cloudwatch from "aws-cdk-lib/aws-cloudwatch";
import * as sns from "aws-cdk-lib/aws-sns";
import * as cloudwatchActions from "aws-cdk-lib/aws-cloudwatch-actions";
import { Construct } from "constructs";

interface MonitoringStackProps extends cdk.StackProps {
  userPoolId: string;
}

export class MonitoringStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props: MonitoringStackProps) {
    super(scope, id, props);

    // アラート通知用 SNS トピック
    const alertTopic = new sns.Topic(this, "SecurityAlertTopic", {
      topicName: "taskflow-security-alerts",
    });

    // 高リスクサインイン試行のアラーム
    const highRiskSignInAlarm = new cloudwatch.Alarm(
      this,
      "HighRiskSignInAlarm",
      {
        alarmName: "TaskFlow-HighRiskSignIn",
        alarmDescription: "高リスクのサインイン試行が検出されました",
        metric: new cloudwatch.Metric({
          namespace: "AWS/Cognito",
          metricName: "RiskScore",
          dimensionsMap: {
            UserPoolId: props.userPoolId,
            RiskLevel: "High",
          },
          statistic: "Sum",
          period: cdk.Duration.minutes(5),
        }),
        threshold: 5, // 5分間に5回以上
        evaluationPeriods: 1,
        comparisonOperator:
          cloudwatch.ComparisonOperator.GREATER_THAN_OR_EQUAL_TO_THRESHOLD,
      }
    );

    highRiskSignInAlarm.addAlarmAction(
      new cloudwatchActions.SnsAction(alertTopic)
    );

    // 侵害された認証情報の検出アラーム
    const compromisedCredentialsAlarm = new cloudwatch.Alarm(
      this,
      "CompromisedCredentialsAlarm",
      {
        alarmName: "TaskFlow-CompromisedCredentials",
        alarmDescription: "侵害された認証情報が検出されました",
        metric: new cloudwatch.Metric({
          namespace: "AWS/Cognito",
          metricName: "CompromisedCredentialsRisk",
          dimensionsMap: {
            UserPoolId: props.userPoolId,
          },
          statistic: "Sum",
          period: cdk.Duration.minutes(5),
        }),
        threshold: 1,
        evaluationPeriods: 1,
        comparisonOperator:
          cloudwatch.ComparisonOperator.GREATER_THAN_OR_EQUAL_TO_THRESHOLD,
      }
    );

    compromisedCredentialsAlarm.addAlarmAction(
      new cloudwatchActions.SnsAction(alertTopic)
    );

    // ダッシュボード
    new cloudwatch.Dashboard(this, "SecurityDashboard", {
      dashboardName: "TaskFlow-Security",
      widgets: [
        [
          new cloudwatch.GraphWidget({
            title: "サインイン試行（リスクレベル別）",
            left: [
              new cloudwatch.Metric({
                namespace: "AWS/Cognito",
                metricName: "SignInSuccesses",
                dimensionsMap: { UserPoolId: props.userPoolId },
                statistic: "Sum",
                period: cdk.Duration.hours(1),
              }),
            ],
          }),
          new cloudwatch.GraphWidget({
            title: "高リスクイベント",
            left: [
              new cloudwatch.Metric({
                namespace: "AWS/Cognito",
                metricName: "RiskScore",
                dimensionsMap: {
                  UserPoolId: props.userPoolId,
                  RiskLevel: "High",
                },
                statistic: "Sum",
                period: cdk.Duration.hours(1),
              }),
            ],
          }),
        ],
      ],
    });
  }
}
```

## 実装タスク

### タスク 1: セキュリティイベントのログ記録

Pre Authentication トリガーでセキュリティイベントをログに記録する機能を実装してください。

<details>
<summary>回答</summary>

```typescript
// infra/lambda/pre-authentication/index.ts
import type {
  PreAuthenticationTriggerEvent,
  PreAuthenticationTriggerHandler,
} from "aws-lambda";
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import { DynamoDBDocumentClient, PutCommand } from "@aws-sdk/lib-dynamodb";

const dynamoClient = new DynamoDBClient({});
const docClient = DynamoDBDocumentClient.from(dynamoClient);

const SECURITY_LOGS_TABLE =
  process.env.SECURITY_LOGS_TABLE || "taskflow-security-logs";

export const handler: PreAuthenticationTriggerHandler = async (event) => {
  const { userName, request, callerContext } = event;
  const { userAttributes } = request;

  // セキュリティイベントをログに記録
  try {
    await docClient.send(
      new PutCommand({
        TableName: SECURITY_LOGS_TABLE,
        Item: {
          PK: `USER#${userName}`,
          SK: `AUTH#${Date.now()}`,
          eventType: "SIGN_IN_ATTEMPT",
          userId: userName,
          email: userAttributes.email,
          clientId: callerContext.clientId,
          timestamp: new Date().toISOString(),
          // 注意: IP アドレスなどの追加情報は Lambda のコンテキストからは取得できない
          // API Gateway 経由の場合は requestContext から取得可能
        },
      })
    );
  } catch (error) {
    console.error("Failed to log security event:", error);
    // ログ失敗でも認証は続行
  }

  return event;
};
```

</details>

## よくある間違い

### ❌ Advanced Security を AUDIT モードのまま本番運用

```typescript
// 悪い例: 監査モードでは実際のブロックが行われない
advancedSecurityMode: cognito.AdvancedSecurityMode.AUDIT,
```

### ✅ 本番環境では ENFORCED モードを使用

```typescript
// 良い例: 強制モードで実際にブロック
advancedSecurityMode: cognito.AdvancedSecurityMode.ENFORCED,
```

## まとめ

この章では以下を学びました：

- Advanced Security の機能と設定方法
- 適応型認証によるリスクベースの認証
- 侵害された認証情報の検出
- CloudWatch によるセキュリティ監視

## 確認クイズ

<details>
<summary>Q1: AUDIT モードと ENFORCED モードの違いは？</summary>

**A1:**

- AUDIT: リスク評価は行うが、実際のブロックや MFA 要求は行わない。ログのみ記録
- ENFORCED: リスク評価に基づいて実際にブロックや MFA 要求を行う

本番環境では ENFORCED を使用し、テスト環境では AUDIT で動作確認することが推奨されます。

</details>

<details>
<summary>Q2: 侵害された認証情報の検出はどのタイミングで行われますか？</summary>

**A2:**
以下のタイミングで検出が行われます：

- サインアップ時
- サインイン時
- パスワード変更時

検出された場合、設定に応じてブロックまたは警告が行われます。

</details>

---

前の章: [10 - Identity Pools と AWS リソース連携](./10-identity-pools.md)
次の章: [12 - 本番環境への展開](./12-production-deployment.md)
