# 10 - Identity Pools と AWS リソース連携

## 学習の目的

- Identity Pools の役割と User Pools との違いを理解する
- 一時的な AWS 認証情報の取得方法を学ぶ
- S3 への直接アクセスを実装する
- IAM ロールとポリシーの設計を学ぶ

## 背景知識

### Identity Pools の役割

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     User Pools vs Identity Pools                         │
│                                                                          │
│  ┌─────────────────────────────────┐  ┌─────────────────────────────┐  │
│  │         User Pools              │  │      Identity Pools         │  │
│  │                                 │  │                             │  │
│  │  "誰であるか" を証明            │  │  "何ができるか" を制御      │  │
│  │                                 │  │                             │  │
│  │  入力: ユーザー名/パスワード    │  │  入力: ID Token             │  │
│  │  出力: JWT トークン             │  │  出力: AWS 認証情報         │  │
│  │        (ID, Access, Refresh)    │  │        (AccessKeyId,        │  │
│  │                                 │  │         SecretAccessKey,    │  │
│  │  用途:                          │  │         SessionToken)       │  │
│  │  • API 認証                     │  │                             │  │
│  │  • ユーザー情報取得             │  │  用途:                      │  │
│  │                                 │  │  • S3 直接アクセス          │  │
│  │                                 │  │  • DynamoDB 直接アクセス    │  │
│  │                                 │  │  • その他 AWS サービス      │  │
│  └─────────────────────────────────┘  └─────────────────────────────┘  │
│                                                                          │
│                              連携フロー                                  │
│                                                                          │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────────────────┐ │
│  │ SignIn  │───▶│ ID Token│───▶│ Identity│───▶│ AWS Credentials     │ │
│  │         │    │         │    │ Pool    │    │                     │ │
│  └─────────┘    └─────────┘    └─────────┘    └─────────────────────┘ │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 認証済み vs 未認証アクセス

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     Identity Pools のアクセスタイプ                      │
│                                                                          │
│  ┌─────────────────────────────────┐  ┌─────────────────────────────┐  │
│  │     認証済みアクセス            │  │    未認証（ゲスト）アクセス │  │
│  │                                 │  │                             │  │
│  │  • User Pool でログイン済み     │  │  • ログインなしでアクセス   │  │
│  │  • 広い権限を付与可能           │  │  • 最小限の権限のみ         │  │
│  │  • ユーザー固有のリソース       │  │  • 公開リソースのみ         │  │
│  │    にアクセス可能               │  │                             │  │
│  │                                 │  │  例:                        │  │
│  │  例:                            │  │  • 公開画像の取得           │  │
│  │  • 自分のファイルをS3に保存     │  │  • アプリ設定の取得         │  │
│  │  • プロフィール画像のアップロード│  │                             │  │
│  └─────────────────────────────────┘  └─────────────────────────────┘  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

## 概念の説明

### IAM ロールの設計

Identity Pools では、認証済みユーザーと未認証ユーザーに異なる IAM ロールを割り当てます：

```typescript
// 認証済みユーザー用ロールのポリシー例
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": [
        "arn:aws:s3:::my-bucket/users/${cognito-identity.amazonaws.com:sub}/*"
      ]
    }
  ]
}
```

`${cognito-identity.amazonaws.com:sub}` は、ユーザーの Identity ID に置換されます。これにより、ユーザーは自分のフォルダにのみアクセスできます。

## コードサンプル

### CDK での Identity Pool 設定

```typescript
// infra/lib/identity-pool-stack.ts
import * as cdk from "aws-cdk-lib";
import * as cognito from "aws-cdk-lib/aws-cognito";
import * as iam from "aws-cdk-lib/aws-iam";
import * as s3 from "aws-cdk-lib/aws-s3";
import { Construct } from "constructs";

interface IdentityPoolStackProps extends cdk.StackProps {
  userPool: cognito.IUserPool;
  userPoolClient: cognito.IUserPoolClient;
}

export class IdentityPoolStack extends cdk.Stack {
  public readonly identityPool: cognito.CfnIdentityPool;
  public readonly uploadBucket: s3.Bucket;

  constructor(scope: Construct, id: string, props: IdentityPoolStackProps) {
    super(scope, id, props);

    // S3 バケット（ユーザーファイル用）
    this.uploadBucket = new s3.Bucket(this, "UserUploadBucket", {
      bucketName: `taskflow-uploads-${this.account}`,
      cors: [
        {
          allowedMethods: [
            s3.HttpMethods.GET,
            s3.HttpMethods.PUT,
            s3.HttpMethods.POST,
          ],
          allowedOrigins: [
            "http://localhost:4200",
            "https://taskflow.example.com",
          ],
          allowedHeaders: ["*"],
        },
      ],
      blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
      encryption: s3.BucketEncryption.S3_MANAGED,
      removalPolicy: cdk.RemovalPolicy.RETAIN,
    });

    // Identity Pool
    this.identityPool = new cognito.CfnIdentityPool(
      this,
      "TaskFlowIdentityPool",
      {
        identityPoolName: "taskflow-identity-pool",
        allowUnauthenticatedIdentities: false, // ゲストアクセスを無効化
        cognitoIdentityProviders: [
          {
            clientId: props.userPoolClient.userPoolClientId,
            providerName: props.userPool.userPoolProviderName,
          },
        ],
      }
    );

    // 認証済みユーザー用 IAM ロール
    const authenticatedRole = new iam.Role(this, "AuthenticatedRole", {
      assumedBy: new iam.FederatedPrincipal(
        "cognito-identity.amazonaws.com",
        {
          StringEquals: {
            "cognito-identity.amazonaws.com:aud": this.identityPool.ref,
          },
          "ForAnyValue:StringLike": {
            "cognito-identity.amazonaws.com:amr": "authenticated",
          },
        },
        "sts:AssumeRoleWithWebIdentity"
      ),
    });

    // S3 アクセスポリシー（ユーザー固有のフォルダのみ）
    authenticatedRole.addToPolicy(
      new iam.PolicyStatement({
        effect: iam.Effect.ALLOW,
        actions: ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
        resources: [
          `${this.uploadBucket.bucketArn}/users/\${cognito-identity.amazonaws.com:sub}/*`,
        ],
      })
    );

    // ユーザーフォルダの一覧取得
    authenticatedRole.addToPolicy(
      new iam.PolicyStatement({
        effect: iam.Effect.ALLOW,
        actions: ["s3:ListBucket"],
        resources: [this.uploadBucket.bucketArn],
        conditions: {
          StringLike: {
            "s3:prefix": ["users/${cognito-identity.amazonaws.com:sub}/*"],
          },
        },
      })
    );

    // ロールのアタッチ
    new cognito.CfnIdentityPoolRoleAttachment(this, "RoleAttachment", {
      identityPoolId: this.identityPool.ref,
      roles: {
        authenticated: authenticatedRole.roleArn,
      },
    });

    // 出力
    new cdk.CfnOutput(this, "IdentityPoolId", {
      value: this.identityPool.ref,
      description: "Identity Pool ID",
    });

    new cdk.CfnOutput(this, "UploadBucketName", {
      value: this.uploadBucket.bucketName,
      description: "S3 Upload Bucket Name",
    });
  }
}
```

### Amplify 設定の更新

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
        identityPoolId: environment.cognito.identityPoolId, // 追加
        signUpVerificationMethod: "code",
      },
    },
    Storage: {
      S3: {
        bucket: environment.s3.uploadBucket,
        region: environment.aws.region,
      },
    },
  });
}
```

### S3 ファイルアップロードサービス

```typescript
// apps/web/src/app/storage/services/storage.service.ts
import { Injectable, inject, signal } from "@angular/core";
import { uploadData, downloadData, remove, list } from "aws-amplify/storage";
import { fetchAuthSession } from "aws-amplify/auth";

export interface UploadProgress {
  loaded: number;
  total: number;
  percentage: number;
}

export interface StorageFile {
  key: string;
  size: number;
  lastModified: Date;
}

@Injectable({
  providedIn: "root",
})
export class StorageService {
  private readonly _uploading = signal(false);
  private readonly _progress = signal<UploadProgress | null>(null);
  private readonly _error = signal<string | null>(null);

  readonly uploading = this._uploading.asReadonly();
  readonly progress = this._progress.asReadonly();
  readonly error = this._error.asReadonly();

  /**
   * ファイルをアップロード
   */
  async uploadFile(file: File, path?: string): Promise<string> {
    this._uploading.set(true);
    this._progress.set(null);
    this._error.set(null);

    try {
      // Identity ID を取得してユーザー固有のパスを生成
      const session = await fetchAuthSession();
      const identityId = session.identityId;

      const key = path
        ? `users/${identityId}/${path}/${file.name}`
        : `users/${identityId}/${file.name}`;

      const result = await uploadData({
        key,
        data: file,
        options: {
          contentType: file.type,
          onProgress: ({ transferredBytes, totalBytes }) => {
            if (totalBytes) {
              this._progress.set({
                loaded: transferredBytes,
                total: totalBytes,
                percentage: Math.round((transferredBytes / totalBytes) * 100),
              });
            }
          },
        },
      }).result;

      return result.key;
    } catch (error) {
      const message =
        error instanceof Error ? error.message : "アップロードに失敗しました";
      this._error.set(message);
      throw error;
    } finally {
      this._uploading.set(false);
    }
  }

  /**
   * ファイルをダウンロード
   */
  async downloadFile(key: string): Promise<Blob> {
    try {
      const result = await downloadData({ key }).result;
      return await result.body.blob();
    } catch (error) {
      const message =
        error instanceof Error ? error.message : "ダウンロードに失敗しました";
      this._error.set(message);
      throw error;
    }
  }

  /**
   * ファイルを削除
   */
  async deleteFile(key: string): Promise<void> {
    try {
      await remove({ key });
    } catch (error) {
      const message =
        error instanceof Error ? error.message : "削除に失敗しました";
      this._error.set(message);
      throw error;
    }
  }

  /**
   * ファイル一覧を取得
   */
  async listFiles(path?: string): Promise<StorageFile[]> {
    try {
      const session = await fetchAuthSession();
      const identityId = session.identityId;

      const prefix = path
        ? `users/${identityId}/${path}/`
        : `users/${identityId}/`;

      const result = await list({
        prefix,
        options: {
          listAll: true,
        },
      });

      return result.items.map((item) => ({
        key: item.key,
        size: item.size || 0,
        lastModified: item.lastModified || new Date(),
      }));
    } catch (error) {
      const message =
        error instanceof Error
          ? error.message
          : "ファイル一覧の取得に失敗しました";
      this._error.set(message);
      throw error;
    }
  }
}
```

### ファイルアップロードコンポーネント

```typescript
// apps/web/src/app/storage/components/file-upload/file-upload.component.ts
import {
  Component,
  inject,
  signal,
  ChangeDetectionStrategy,
} from "@angular/core";
import { CommonModule } from "@angular/common";
import {
  StorageService,
  type StorageFile,
} from "../../services/storage.service";

@Component({
  selector: "app-file-upload",
  standalone: true,
  imports: [CommonModule],
  template: `
    <div class="upload-container">
      <h2>ファイル管理</h2>

      @if (storageService.error()) {
      <div class="error-message">{{ storageService.error() }}</div>
      }

      <!-- アップロードエリア -->
      <div
        class="upload-area"
        [class.dragging]="isDragging()"
        (dragover)="onDragOver($event)"
        (dragleave)="onDragLeave($event)"
        (drop)="onDrop($event)"
      >
        <input
          type="file"
          #fileInput
          (change)="onFileSelected($event)"
          style="display: none"
          multiple
        />

        @if (storageService.uploading()) {
        <div class="progress">
          <p>アップロード中... {{ storageService.progress()?.percentage }}%</p>
          <div class="progress-bar">
            <div
              class="progress-fill"
              [style.width.%]="storageService.progress()?.percentage"
            ></div>
          </div>
        </div>
        } @else {
        <p>ファイルをドラッグ＆ドロップ</p>
        <p>または</p>
        <button (click)="fileInput.click()">ファイルを選択</button>
        }
      </div>

      <!-- ファイル一覧 -->
      <div class="file-list">
        <h3>アップロード済みファイル</h3>

        @if (loading()) {
        <p>読み込み中...</p>
        } @else if (files().length === 0) {
        <p class="no-files">ファイルがありません</p>
        } @else {
        <ul>
          @for (file of files(); track file.key) {
          <li class="file-item">
            <span class="file-name">{{ getFileName(file.key) }}</span>
            <span class="file-size">{{ formatSize(file.size) }}</span>
            <div class="file-actions">
              <button (click)="downloadFile(file)">ダウンロード</button>
              <button class="btn-danger" (click)="deleteFile(file)">
                削除
              </button>
            </div>
          </li>
          }
        </ul>
        }
      </div>
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class FileUploadComponent {
  readonly storageService = inject(StorageService);

  readonly isDragging = signal(false);
  readonly loading = signal(true);
  readonly files = signal<StorageFile[]>([]);

  async ngOnInit(): Promise<void> {
    await this.loadFiles();
  }

  async loadFiles(): Promise<void> {
    this.loading.set(true);
    try {
      const files = await this.storageService.listFiles();
      this.files.set(files);
    } finally {
      this.loading.set(false);
    }
  }

  onDragOver(event: DragEvent): void {
    event.preventDefault();
    this.isDragging.set(true);
  }

  onDragLeave(event: DragEvent): void {
    event.preventDefault();
    this.isDragging.set(false);
  }

  async onDrop(event: DragEvent): Promise<void> {
    event.preventDefault();
    this.isDragging.set(false);

    const files = event.dataTransfer?.files;
    if (files) {
      await this.uploadFiles(Array.from(files));
    }
  }

  async onFileSelected(event: Event): Promise<void> {
    const input = event.target as HTMLInputElement;
    if (input.files) {
      await this.uploadFiles(Array.from(input.files));
    }
  }

  async uploadFiles(files: File[]): Promise<void> {
    for (const file of files) {
      try {
        await this.storageService.uploadFile(file);
      } catch {
        // エラーは StorageService で処理済み
      }
    }
    await this.loadFiles();
  }

  async downloadFile(file: StorageFile): Promise<void> {
    try {
      const blob = await this.storageService.downloadFile(file.key);
      const url = URL.createObjectURL(blob);
      const a = document.createElement("a");
      a.href = url;
      a.download = this.getFileName(file.key);
      a.click();
      URL.revokeObjectURL(url);
    } catch {
      // エラーは StorageService で処理済み
    }
  }

  async deleteFile(file: StorageFile): Promise<void> {
    if (!confirm("このファイルを削除しますか？")) return;

    try {
      await this.storageService.deleteFile(file.key);
      await this.loadFiles();
    } catch {
      // エラーは StorageService で処理済み
    }
  }

  getFileName(key: string): string {
    return key.split("/").pop() || key;
  }

  formatSize(bytes: number): string {
    if (bytes < 1024) return `${bytes} B`;
    if (bytes < 1024 * 1024) return `${(bytes / 1024).toFixed(1)} KB`;
    return `${(bytes / (1024 * 1024)).toFixed(1)} MB`;
  }
}
```

## 実装タスク

### タスク 1: プロフィール画像アップロード機能

ユーザーのプロフィール画像をアップロードする機能を実装してください。

<details>
<summary>回答</summary>

```typescript
// apps/web/src/app/profile/services/profile.service.ts
import { Injectable, inject } from "@angular/core";
import { StorageService } from "../../storage/services/storage.service";
import { AuthService } from "../../auth/services/auth.service";

@Injectable({
  providedIn: "root",
})
export class ProfileService {
  private readonly storageService = inject(StorageService);
  private readonly authService = inject(AuthService);

  async uploadProfileImage(file: File): Promise<string> {
    // 画像ファイルのバリデーション
    if (!file.type.startsWith("image/")) {
      throw new Error("画像ファイルを選択してください");
    }

    // ファイルサイズ制限（5MB）
    if (file.size > 5 * 1024 * 1024) {
      throw new Error("ファイルサイズは5MB以下にしてください");
    }

    // プロフィール画像用のパスにアップロード
    const key = await this.storageService.uploadFile(file, "profile");
    return key;
  }
}
```

</details>

## よくある間違い

### ❌ Identity Pool なしで S3 に直接アクセス

```typescript
// 悪い例: API 経由でファイルをアップロード（非効率）
const formData = new FormData();
formData.append("file", file);
await fetch("/api/upload", { method: "POST", body: formData });
```

### ✅ Identity Pool で一時認証情報を取得して直接アクセス

```typescript
// 良い例: S3 に直接アップロード（効率的）
await uploadData({ key, data: file }).result;
```

## まとめ

この章では以下を学びました：

- Identity Pools と User Pools の違い
- 一時的な AWS 認証情報の取得
- S3 への直接アクセス実装
- ユーザー固有のリソースアクセス制御

## 確認クイズ

<details>
<summary>Q1: Identity Pools を使う利点は何ですか？</summary>

**A1:**

1. サーバーを経由せずに AWS リソースに直接アクセスできる
2. 一時的な認証情報なのでセキュリティが高い
3. ユーザーごとに異なる権限を IAM ポリシーで制御できる
4. サーバーの負荷を軽減できる
</details>

<details>
<summary>Q2: `${cognito-identity.amazonaws.com:sub}` とは何ですか？</summary>

**A2:**
Identity Pool が各ユーザーに割り当てる一意の識別子（Identity ID）です。IAM ポリシーでこの変数を使用することで、ユーザーごとに異なるリソースへのアクセスを制御できます。

例えば、S3 バケットで `users/${cognito-identity.amazonaws.com:sub}/*` と指定すると、各ユーザーは自分のフォルダにのみアクセスできます。

</details>

---

前の章: [09 - Lambda Triggers によるカスタマイズ](./09-lambda-triggers.md)
次の章: [11 - Advanced Security](./11-advanced-security.md)
