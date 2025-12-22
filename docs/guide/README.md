# Amazon Cognito 実践学習ガイド

マルチテナント型タスク管理アプリ「TaskFlow」を構築しながら、Amazon Cognito の実務レベルの機能を網羅的に学習するガイドです。

## 対象読者

- AWS の基本的な使用経験がある方
- 認証システムの実装経験がない方
- TypeScript / Angular の基礎を理解している方

## 技術スタック

- フロントエンド: Angular 19+ / TypeScript
- バックエンド: Hono / TypeScript
- インフラ: AWS CDK / TypeScript
- 認証: Amazon Cognito (User Pools / Identity Pools)

## 目次

| 章                                   | タイトル                           | 内容                                                 |
| ------------------------------------ | ---------------------------------- | ---------------------------------------------------- |
| [00](./00-overview.md)               | 概要                               | 全体像、アーキテクチャ、プロジェクト構成             |
| [01](./01-cognito-basics.md)         | Cognito の基礎と環境構築           | User Pools の概念、CDK によるインフラ構築            |
| [02](./02-basic-auth-flow.md)        | 基本認証フローの実装               | サインアップ、サインイン、メール検証                 |
| [03](./03-angular-auth-ui.md)        | Angular 認証 UI の構築             | AuthGuard、HTTP Interceptor、状態管理                |
| [04](./04-hono-api-jwt.md)           | Hono API と JWT 検証               | JWT 検証ミドルウェア、API Gateway 連携               |
| [05](./05-mfa-implementation.md)     | MFA（多要素認証）の実装            | TOTP、SMS、Email OTP                                 |
| [06](./06-passkey-authentication.md) | パスキー認証の実装                 | WebAuthn/FIDO2 によるパスワードレス認証              |
| [07](./07-social-login.md)           | ソーシャルログインの統合           | Google ログイン、Hosted UI                           |
| [08](./08-groups-permissions.md)     | グループと権限管理                 | Cognito Groups、RBAC                                 |
| [09](./09-lambda-triggers.md)        | Lambda Triggers によるカスタマイズ | Pre Sign-up、Post Confirmation、Pre Token Generation |
| [10](./10-identity-pools.md)         | Identity Pools と AWS リソース連携 | 一時的な AWS 認証情報、S3 アクセス                   |
| [11](./11-advanced-security.md)      | Advanced Security                  | 適応型認証、侵害された認証情報の検出                 |
| [12](./12-production-deployment.md)  | 本番環境への展開                   | セキュリティベストプラクティス、監視、運用           |

## 学習の進め方

1. [00-overview.md](./00-overview.md) で全体像を把握
2. 各章を順番に進める
3. 実装タスクを実際にコーディング
4. 確認クイズで理解度をチェック

## 作成するアプリケーション

「TaskFlow」- マルチテナント型タスク管理アプリ

- 組織（テナント）ごとにタスクを管理
- Admin / Member ロールによる権限管理
- パスワード、パスキー、Google ログイン、MFA に対応

## 参考リンク

- [Amazon Cognito Developer Guide](https://docs.aws.amazon.com/cognito/latest/developerguide/)
- [AWS Amplify Auth Documentation](https://docs.amplify.aws/lib/auth/getting-started/)
- [Angular Documentation](https://angular.dev/)
- [Hono Documentation](https://hono.dev/)
