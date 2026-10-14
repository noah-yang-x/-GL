# AWS 学習ロードマップ

**補足：** Device Management Platformは基本的な機能から開発を始め、AWSの学習が深まるにつれて、機能を段階的に追加・拡張していきます。

```mermaid
flowchart TB
    subgraph P1["第1段階：Webフロントエンドエンジニア → Cloud Full-stack Engineer"]
        direction TB
        Project["Device Management Platform"]
        Main["メインルート：デバイス一覧を表示<br/>Browser → AWS → DB → Browser"]
        CRUD["メインルートの拡張：Device CRUD<br/>一覧表示 / 個別表示 / 作成 / 更新 / 削除"]
        Branch["追加機能：ログイン、ファイル、非同期処理、イベント"]
        Auth["認証<br/>Cognito / JWT / API認証・認可"]
        File["ファイル<br/>S3 / Presigned URL / CloudFront"]
        Async["非同期処理<br/>SQS / Worker Lambda / Retry / DLQ"]
        Event["イベント<br/>EventBridge / Audit Log / 管理者への通知"]
        Thick["メインルートの強化：セキュリティ、ネットワーク、監視<br/>IAM / VPC / CloudWatch / WAF / KMS<br/>Secrets Manager / HTTPS / S3 Private / CloudFront OAC"]
        Backend["アーキテクチャの拡張：2つ目のバックエンドを構築<br/>Version A：API Gateway → Lambda → DynamoDB<br/>Version B：CloudFront → ALB → ECS/Fargate<br/>→ Node.js Backend → RDS/PostgreSQL"]
        Pro["実践的な開発・運用体制の整備<br/>IaC / AWS CDK / CI/CD / Test<br/>dev / staging / production<br/>Rollback / Backup / Monitoring / Alarm / Audit"]
        Project --> Main --> CRUD --> Branch
        Branch --> Auth
        Branch --> File
        Branch --> Async
        Branch --> Event
        Auth --> Thick
        File --> Thick
        Async --> Thick
        Event --> Thick
        Thick --> Backend --> Pro
    end

    subgraph P2["第2段階：IoT Cloud"]
        direction TB
        Device["実機 / シミュレーションによるIoTデバイス"]
        MQTT["MQTT"]
        IoT["AWS IoT Core"]
        Existing["第1段階で構築したCloud Backend<br/>↓<br/>第1段階で構築したWeb"]
        Device --> MQTT --> IoT --> Existing
    end

    Pro --> Device
```

## 重要事項

- **同じプロジェクト**：Device Management Platformを第1段階から第2段階へ拡張する。
- **図の矢印**：学習の順序を示す。第1段階はメインルートを動かしてから追加機能を組み込む。
- **第1段階**：ダミーのデバイスデータを使用する。実機・シミュレーションによるデバイスは第2段階で接続する。
- **進度と費用**：6か月以内に学習を終える期限は設けない。費用はできるだけ抑え、有料サービスの利用前に料金を説明する。

原文：[[AWS学习路线]]。
