# AWS構築パラメータシート v2.0

**案件名:** eラーニングシステム構築
**作成日:** 2025年11月22日
**対象リージョン:** ap-northeast-1 (東京)
**環境:** 本番環境 (Production)

---

## 📋 目次

1. [プロジェクト基本情報](#1-プロジェクト基本情報)
2. [VPCネットワーク構成](#2-vpcネットワーク構成)
3. [EC2 & Auto Scaling](#3-ec2--auto-scaling)
4. [RDS Aurora MySQL](#4-rds-aurora-mysql)
5. [Application Load Balancer (ALB)](#5-application-load-balancer-alb)
6. [S3バケット](#6-s3バケット)
7. [セキュリティグループ](#7-セキュリティグループ)
8. [IAMロール・ポリシー](#8-iamロールポリシー)
9. [Route 53 & ACM](#9-route-53--acm)
10. [CloudWatch監視・アラート](#10-cloudwatch監視アラート)
11. [WAF設定](#11-waf設定)
12. [バックアップ設定](#12-バックアップ設定)

---

## 1. プロジェクト基本情報

### 1.1 基本設定

```yaml
プロジェクト名: kawausolms
環境識別子: prod
リージョン: ap-northeast-1
使用AZ:
  - ap-northeast-1a
  - ap-northeast-1c
```

### 1.2 命名規則

**パターン:** `{project}-{env}-{resource}-{detail}`

**例:**
- VPC: `kawausolms-prod-vpc`
- Subnet: `kawausolms-prod-subnet-public-1a`
- EC2: `kawausolms-prod-lt-ec2`

---

## 2. VPCネットワーク構成

### 2.1 VPC設定

| パラメータ      | 設定値                          |
| :-------------- | :------------------------------ |
| **VPC名**       | `kawausolms-prod-vpc`           |
| **CIDR**        | `10.0.0.0/16`                   |
| **DNS解決**     | 有効 (enableDnsSupport: true)   |
| **DNSホスト名** | 有効 (enableDnsHostnames: true) |
| **IPv6**        | 無効                            |

### 2.2 サブネット構成

#### パブリックサブネット

| サブネット名                       | AZ              | CIDR          | 用途             |
| :--------------------------------- | :-------------- | :------------ | :--------------- |
| `kawausolms-prod-subnet-public-1a` | ap-northeast-1a | `10.0.1.0/24` | ALB, NAT Gateway |
| `kawausolms-prod-subnet-public-1c` | ap-northeast-1c | `10.0.2.0/24` | ALB, NAT Gateway |

**設定:**
- パブリックIP自動割り当て: **無効**
- インターネットゲートウェイ: `kawausolms-prod-igw`

#### プライベートサブネット (App層)

| サブネット名                            | AZ              | CIDR           | 用途            |
| :-------------------------------------- | :-------------- | :------------- | :-------------- |
| `kawausolms-prod-subnet-private-app-1a` | ap-northeast-1a | `10.0.11.0/24` | EC2インスタンス |
| `kawausolms-prod-subnet-private-app-1c` | ap-northeast-1c | `10.0.12.0/24` | EC2インスタンス |

**設定:**
- NAT Gateway経由でインターネットアクセス

#### プライベートサブネット (DB層)

| サブネット名                           | AZ              | CIDR           | 用途       |
| :------------------------------------- | :-------------- | :------------- | :--------- |
| `kawausolms-prod-subnet-private-db-1a` | ap-northeast-1a | `10.0.21.0/24` | RDS Aurora |
| `kawausolms-prod-subnet-private-db-1c` | ap-northeast-1c | `10.0.22.0/24` | RDS Aurora |

### 2.3 ゲートウェイ・ルーティング

#### インターネットゲートウェイ

| パラメータ        | 設定値                |
| :---------------- | :-------------------- |
| **IGW名**         | `kawausolms-prod-igw` |
| **アタッチ先VPC** | `kawausolms-prod-vpc` |

#### NAT Gateway

| NAT Gateway名              | AZ              | サブネット                         | Elastic IP |
| :------------------------- | :-------------- | :--------------------------------- | :--------- |
| `kawausolms-prod-natgw-1a` | ap-northeast-1a | `kawausolms-prod-subnet-public-1a` | 新規作成   |
| `kawausolms-prod-natgw-1c` | ap-northeast-1c | `kawausolms-prod-subnet-public-1c` | 新規作成   |

#### ルートテーブル

**パブリックルートテーブル:** `kawausolms-prod-rtb-public`

| 送信先      | ターゲット            |
| :---------- | :-------------------- |
| 10.0.0.0/16 | local                 |
| 0.0.0.0/0   | `kawausolms-prod-igw` |

**プライベートルートテーブル (AZ-1a):** `kawausolms-prod-rtb-private-1a`

| 送信先      | ターゲット                 |
| :---------- | :------------------------- |
| 10.0.0.0/16 | local                      |
| 0.0.0.0/0   | `kawausolms-prod-natgw-1a` |

**プライベートルートテーブル (AZ-1c):** `kawausolms-prod-rtb-private-1c`

| 送信先      | ターゲット                 |
| :---------- | :------------------------- |
| 10.0.0.0/16 | local                      |
| 0.0.0.0/0   | `kawausolms-prod-natgw-1c` |

---

## 3. EC2 & Auto Scaling

### 3.1 Launch Template設定

| パラメータ             | 設定値                                 |
| :--------------------- | :------------------------------------- |
| **Launch Template名**  | `kawausolms-prod-lt-ec2`               |
| **AMI**                | Amazon Linux 2023 (最新)               |
| **インスタンスタイプ** | `t3.medium` (2vCPU, 4GB RAM)           |
| **キーペア**           | 運用時に設定 (SSH予備用)               |
| **IAMロール**          | `kawausolms-prod-role-ec2`             |
| **ユーザーデータ**     | アプリケーションセットアップスクリプト |

#### EBSボリューム設定

| パラメータ           | 設定値                |
| :------------------- | :-------------------- |
| **ボリュームタイプ** | gp3                   |
| **サイズ**           | 20 GB                 |
| **IOPS**             | 3,000 (デフォルト)    |
| **スループット**     | 125 MB/s (デフォルト) |
| **暗号化**           | 有効 (AWS KMS)        |
| **終了時削除**       | 有効 (true)           |

#### ネットワーク設定

| パラメータ               | 設定値                   |
| :----------------------- | :----------------------- |
| **セキュリティグループ** | `kawausolms-prod-sg-ec2` |
| **パブリックIP**         | 無効                     |

### 3.2 Auto Scaling Group設定

| パラメータ                 | 設定値                                                                             |
| :------------------------- | :--------------------------------------------------------------------------------- |
| **Auto Scaling Group名**   | `kawausolms-prod-asg-ec2`                                                          |
| **Launch Template**        | `kawausolms-prod-lt-ec2` (最新バージョン)                                          |
| **最小キャパシティ**       | 2                                                                                  |
| **希望キャパシティ**       | 2                                                                                  |
| **最大キャパシティ**       | 10                                                                                 |
| **サブネット**             | `kawausolms-prod-subnet-private-app-1a`<br>`kawausolms-prod-subnet-private-app-1c` |
| **ヘルスチェックタイプ**   | ELB                                                                                |
| **ヘルスチェック猶予期間** | 300秒                                                                              |
| **ターゲットグループ**     | `kawausolms-prod-tg-ec2`                                                           |

#### スケーリングポリシー

**スケールアウトポリシー:**

| パラメータ       | 設定値                             |
| :--------------- | :--------------------------------- |
| **ポリシー名**   | `kawausolms-prod-scale-out-policy` |
| **メトリクス**   | CPU使用率                          |
| **閾値**         | 70%                                |
| **評価期間**     | 2回連続 × 1分                      |
| **アクション**   | +1インスタンス追加                 |
| **クールダウン** | 300秒                              |

**スケールインポリシー:**

| パラメータ       | 設定値                            |
| :--------------- | :-------------------------------- |
| **ポリシー名**   | `kawausolms-prod-scale-in-policy` |
| **メトリクス**   | CPU使用率                         |
| **閾値**         | 30%                               |
| **評価期間**     | 2回連続 × 5分                     |
| **アクション**   | -1インスタンス削除                |
| **クールダウン** | 300秒                             |

### 3.3 アプリケーション構成

| 項目             | 設定値                               |
| :--------------- | :----------------------------------- |
| **OS**           | Amazon Linux 2023                    |
| **言語**         | Java                                 |
| **Webサーバー**  | Nginx                                |
| **アプリ容量**   | 約5GB                                |
| **デプロイ方式** | ゴールデンイメージ (AMI)             |
| **AMI更新頻度**  | アプリ更新時                         |
| **AMI保持世代**  | 4世代                                |
| **パッチ適用**   | 月次 (Systems Manager Patch Manager) |

---

## 4. RDS Aurora MySQL

### 4.1 クラスター設定

| パラメータ               | 設定値                                                                           |
| :----------------------- | :------------------------------------------------------------------------------- |
| **クラスター識別子**     | `kawausolms-prod-aurora-cluster`                                                 |
| **エンジン**             | Aurora MySQL 3.x (MySQL 8.0互換)                                                 |
| **エンジンバージョン**   | 3.x系最新                                                                        |
| **マスターユーザー名**   | `admin`                                                                          |
| **マスターパスワード**   | AWS Secrets Manager管理 (緊急時のみ使用)                                         |
| **DBサブネットグループ** | `kawausolms-prod-subnet-private-db-1a`<br>`kawausolms-prod-subnet-private-db-1c` |
| **セキュリティグループ** | `kawausolms-prod-sg-rds`                                                         |
| **暗号化**               | 有効 (AWS KMS)                                                                   |
| **削除保護**             | 有効                                                                             |

### 4.2 Writerインスタンス設定

| パラメータ             | 設定値                          |
| :--------------------- | :------------------------------ |
| **インスタンス識別子** | `kawausolms-prod-aurora-writer` |
| **インスタンスクラス** | `db.t3.large` (2vCPU, 8GB RAM)  |
| **AZ**                 | ap-northeast-1a                 |
| **パブリックアクセス** | 無効                            |
| **拡張モニタリング**   | 有効 (60秒間隔)                 |

### 4.3 Readerインスタンス設定 (Auto Scaling)

| パラメータ               | 設定値                         |
| :----------------------- | :----------------------------- |
| **最小Reader数**         | 1                              |
| **最大Reader数**         | 3                              |
| **インスタンスクラス**   | `db.t3.large` (2vCPU, 8GB RAM) |
| **スケールアウト閾値**   | CPU 70%                        |
| **スケールイン閾値**     | CPU 30%                        |
| **クールダウン時間**     | 300秒                          |
| **ターゲットメトリクス** | CPUUtilization                 |

**Readerインスタンス名:**
- `kawausolms-prod-aurora-reader-1` (最小構成)
- `kawausolms-prod-aurora-reader-2` (Auto Scaling)
- `kawausolms-prod-aurora-reader-3` (Auto Scaling)

### 4.4 ストレージ設定

| パラメータ           | 設定値            |
| :------------------- | :---------------- |
| **初期ストレージ**   | 10 GB             |
| **最大ストレージ**   | 128 TB (自動拡張) |
| **ストレージタイプ** | Aurora標準 (SSD)  |
| **暗号化**           | 有効 (AWS KMS)    |

### 4.5 バックアップ設定

| パラメータ                     | 設定値                   |
| :----------------------------- | :----------------------- |
| **自動バックアップ**           | 有効                     |
| **バックアップ保持期間**       | 7日間                    |
| **バックアップウィンドウ**     | 02:00-03:00 JST          |
| **ポイントインタイムリカバリ** | 有効 (5分以内の任意時点) |
| **スナップショット**           | 手動取得可能             |

### 4.6 メンテナンス設定

| パラメータ                               | 設定値                 |
| :--------------------------------------- | :--------------------- |
| **メンテナンスウィンドウ**               | 土曜日 03:00-04:00 JST |
| **自動マイナーバージョンアップグレード** | 無効 (手動管理)        |

### 4.7 ログ設定

| ログタイプ           | 有効/無効        | 保存先 | 保持期間 |
| :------------------- | :--------------- | :----- | :------- |
| **エラーログ**       | 有効             | S3     | 90日     |
| **スロークエリログ** | 有効 (閾値: 2秒) | S3     | 90日     |
| **監査ログ**         | 無効             | -      | -        |
| **一般ログ**         | 無効             | -      | -        |

### 4.8 認証・接続設定

| パラメータ             | 設定値                    |
| :--------------------- | :------------------------ |
| **認証方法**           | パスワード認証            |
| **IAM認証**            | 無効                      |
| **マスターユーザー**   | `admin` (緊急時のみ)      |
| **アプリ用ユーザー**   | RDS内で個別作成           |
| **パスワード管理**     | AWS Secrets Manager       |
| **自動ローテーション** | 有効 (90日、アプリ用のみ) |

### 4.9 パフォーマンス設定

| パラメータ                  | 設定値              |
| :-------------------------- | :------------------ |
| **パラメータグループ**      | カスタム作成        |
| **max_connections**         | 400                 |
| **innodb_buffer_pool_size** | 5GB (メモリの約60%) |
| **slow_query_log**          | 1 (有効)            |
| **long_query_time**         | 2 (秒)              |

### 4.10 負荷試算データ

| 項目                         | 想定値               |
| :--------------------------- | :------------------- |
| **ピーク時同時接続ユーザー** | 1,800人              |
| **総クエリ数 (ピーク時)**    | 300 QPS              |
| **読み取りクエリ (80%)**     | 240 QPS → Reader分散 |
| **書き込みクエリ (20%)**     | 60 QPS → Writer処理  |
| **平均クエリ実行時間**       | 10〜50ms             |
| **接続プールサイズ**         | 200〜400接続         |
| **CPU使用率 (通常時)**       | 30〜40%              |
| **CPU使用率 (ピーク時)**     | 50〜60%              |
| **メモリ使用率**             | 60〜80%              |

---

## 5. Application Load Balancer (ALB)

### 5.1 ALB基本設定

| パラメータ               | 設定値                                                                   |
| :----------------------- | :----------------------------------------------------------------------- |
| **ALB名**                | `kawausolms-prod-alb`                                                    |
| **スキーム**             | internet-facing                                                          |
| **IPアドレスタイプ**     | IPv4                                                                     |
| **サブネット**           | `kawausolms-prod-subnet-public-1a`<br>`kawausolms-prod-subnet-public-1c` |
| **セキュリティグループ** | `kawausolms-prod-sg-alb`                                                 |
| **削除保護**             | 有効                                                                     |

### 5.2 リスナー設定

#### HTTPSリスナー (Port 443)

| パラメータ               | 設定値                              |
| :----------------------- | :---------------------------------- |
| **プロトコル**           | HTTPS                               |
| **ポート**               | 443                                 |
| **SSL証明書**            | ACM証明書 (*.kawausolms.com)        |
| **セキュリティポリシー** | ELBSecurityPolicy-TLS13-1-2-2021-06 |
| **デフォルトアクション** | Forward to `kawausolms-prod-tg-ec2` |

#### HTTPリスナー (Port 80)

| パラメータ               | 設定値                    |
| :----------------------- | :------------------------ |
| **プロトコル**           | HTTP                      |
| **ポート**               | 80                        |
| **デフォルトアクション** | **ブロック (設定しない)** |

**注:** HTTPアクセスは許可しない方針

### 5.3 ターゲットグループ設定

| パラメータ               | 設定値                   |
| :----------------------- | :----------------------- |
| **ターゲットグループ名** | `kawausolms-prod-tg-ec2` |
| **ターゲットタイプ**     | instance                 |
| **プロトコル**           | HTTP                     |
| **ポート**               | 80                       |
| **VPC**                  | `kawausolms-prod-vpc`    |
| **プロトコルバージョン** | HTTP1                    |

#### ヘルスチェック設定

| パラメータ       | 設定値                     |
| :--------------- | :------------------------- |
| **プロトコル**   | HTTP                       |
| **パス**         | `/health` (アプリ側で実装) |
| **ポート**       | traffic-port (80)          |
| **正常閾値**     | 2回連続成功                |
| **非正常閾値**   | 2回連続失敗                |
| **タイムアウト** | 5秒                        |
| **間隔**         | 30秒                       |
| **成功コード**   | 200                        |

#### ターゲットグループ属性

| パラメータ                 | 設定値                |
| :------------------------- | :-------------------- |
| **登録解除の遅延**         | 300秒                 |
| **スティッキーセッション** | 有効 (Cookie: AWSALB) |
| **スティッキー期間**       | 86400秒 (24時間)      |

### 5.4 アクセスログ設定

| パラメータ         | 設定値                                  |
| :----------------- | :-------------------------------------- |
| **アクセスログ**   | 有効                                    |
| **S3バケット**     | `kawausolms-prod-alb-logs-{account-id}` |
| **プレフィックス** | `alb-logs/`                             |

---

## 6. S3バケット

### 6.1 教材コンテンツバケット

| パラメータ             | 設定値                                  |
| :--------------------- | :-------------------------------------- |
| **バケット名**         | `kawausolms-prod-contents-{account-id}` |
| **リージョン**         | ap-northeast-1                          |
| **バージョニング**     | 有効 (最新バージョンのみ保持)           |
| **暗号化**             | SSE-S3 (AES-256)                        |
| **パブリックアクセス** | ブロック (全て)                         |
| **オブジェクトロック** | 無効                                    |

#### ライフサイクルポリシー

| ルール名              | 対象             | アクション |
| :-------------------- | :--------------- | :--------- |
| `delete-old-versions` | 非現行バージョン | 即時削除   |

#### バケットポリシー

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowEC2Access",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::{account-id}:role/kawausolms-prod-role-ec2"
      },
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::kawausolms-prod-contents-{account-id}/*"
    }
  ]
}
```

### 6.2 アプリケーションログバケット

| パラメータ             | 設定値                              |
| :--------------------- | :---------------------------------- |
| **バケット名**         | `kawausolms-prod-logs-{account-id}` |
| **リージョン**         | ap-northeast-1                      |
| **バージョニング**     | 無効                                |
| **暗号化**             | SSE-S3 (AES-256)                    |
| **パブリックアクセス** | ブロック (全て)                     |

#### ライフサイクルポリシー

| ルール名                 | 対象         | アクション   |
| :----------------------- | :----------- | :----------- |
| `expire-logs-90days`     | `app-logs/*` | 90日後に削除 |
| `expire-rds-logs-90days` | `rds-logs/*` | 90日後に削除 |

#### フォルダ構造

```
kawausolms-prod-logs-{account-id}/
├── app-logs/
│   ├── YYYY/MM/DD/
│   └── application.log
└── rds-logs/
    ├── error/
    └── slowquery/
```

### 6.3 ALBアクセスログバケット

| パラメータ             | 設定値                                  |
| :--------------------- | :-------------------------------------- |
| **バケット名**         | `kawausolms-prod-alb-logs-{account-id}` |
| **リージョン**         | ap-northeast-1                          |
| **バージョニング**     | 無効                                    |
| **暗号化**             | SSE-S3 (AES-256)                        |
| **パブリックアクセス** | ブロック (全て)                         |

#### ライフサイクルポリシー

| ルール名                 | 対象         | アクション   |
| :----------------------- | :----------- | :----------- |
| `expire-alb-logs-90days` | `alb-logs/*` | 90日後に削除 |

#### バケットポリシー (ALB書き込み許可)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AWSLogDeliveryWrite",
      "Effect": "Allow",
      "Principal": {
        "Service": "elasticloadbalancing.amazonaws.com"
      },
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::kawausolms-prod-alb-logs-{account-id}/alb-logs/*"
    }
  ]
}
```

---

## 7. セキュリティグループ

### 7.1 ALB用セキュリティグループ

**セキュリティグループ名:** `kawausolms-prod-sg-alb`
**VPC:** `kawausolms-prod-vpc`

#### インバウンドルール

| タイプ | プロトコル | ポート | ソース    | 説明                              |
| :----- | :--------- | :----- | :-------- | :-------------------------------- |
| HTTPS  | TCP        | 443    | 0.0.0.0/0 | インターネットからのHTTPSアクセス |

#### アウトバウンドルール

| タイプ | プロトコル | ポート | 送信先                   | 説明            |
| :----- | :--------- | :----- | :----------------------- | :-------------- |
| HTTP   | TCP        | 80     | `kawausolms-prod-sg-ec2` | EC2へのHTTP転送 |

### 7.2 EC2用セキュリティグループ

**セキュリティグループ名:** `kawausolms-prod-sg-ec2`
**VPC:** `kawausolms-prod-vpc`

#### インバウンドルール

| タイプ | プロトコル | ポート | ソース                   | 説明                       |
| :----- | :--------- | :----- | :----------------------- | :------------------------- |
| HTTP   | TCP        | 80     | `kawausolms-prod-sg-alb` | ALBからのHTTPアクセス      |
| SSH    | TCP        | 22     | `{開発会社固定IP}/32`    | 緊急時のSSHアクセス (予備) |

**注:** SSH接続元IPは運用時に設定

#### アウトバウンドルール

| タイプ       | プロトコル | ポート | 送信先                   | 説明            |
| :----------- | :--------- | :----- | :----------------------- | :-------------- |
| HTTPS        | TCP        | 443    | 0.0.0.0/0                | AWS API、OS更新 |
| MySQL/Aurora | TCP        | 3306   | `kawausolms-prod-sg-rds` | RDSへのDB接続   |

### 7.3 RDS用セキュリティグループ

**セキュリティグループ名:** `kawausolms-prod-sg-rds`
**VPC:** `kawausolms-prod-vpc`

#### インバウンドルール

| タイプ       | プロトコル | ポート | ソース                   | 説明            |
| :----------- | :--------- | :----- | :----------------------- | :-------------- |
| MySQL/Aurora | TCP        | 3306   | `kawausolms-prod-sg-ec2` | EC2からのDB接続 |

#### アウトバウンドルール

なし (RDSからの外部接続は不要)

---

## 8. IAMロール・ポリシー

### 8.1 EC2用IAMロール

**ロール名:** `kawausolms-prod-role-ec2`

#### 信頼関係ポリシー

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

#### アタッチするAWS管理ポリシー

| ポリシー名                     | 用途                          |
| :----------------------------- | :---------------------------- |
| `AmazonSSMManagedInstanceCore` | Systems Manager経由のアクセス |

#### カスタムインラインポリシー

**ポリシー名:** `kawausolms-prod-ec2-custom-policy`

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3ContentsAccess",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::kawausolms-prod-contents-*",
        "arn:aws:s3:::kawausolms-prod-contents-*/*"
      ]
    },
    {
      "Sid": "S3LogsAccess",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject"
      ],
      "Resource": [
        "arn:aws:s3:::kawausolms-prod-logs-*/app-logs/*"
      ]
    },
    {
      "Sid": "CloudWatchLogsAccess",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:DescribeLogStreams"
      ],
      "Resource": [
        "arn:aws:logs:ap-northeast-1:*:log-group:/aws/kawausolms/prod/*"
      ]
    },
    {
      "Sid": "SecretsManagerAccess",
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": [
        "arn:aws:secretsmanager:ap-northeast-1:*:secret:kawausolms-prod-db-app-user-*"
      ]
    }
  ]
}
```

### 8.2 RDS拡張モニタリング用IAMロール

**ロール名:** `kawausolms-prod-role-rds-monitoring`

#### 信頼関係ポリシー

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "monitoring.rds.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

#### アタッチするAWS管理ポリシー

| ポリシー名                        | 用途                |
| :-------------------------------- | :------------------ |
| `AmazonRDSEnhancedMonitoringRole` | RDS拡張モニタリング |

---

## 9. Route 53 & ACM

### 9.1 Route 53ホストゾーン

| パラメータ             | 設定値           |
| :--------------------- | :--------------- |
| **ドメイン名**         | `kawausolms.com` |
| **ホストゾーンタイプ** | パブリック       |
| **ホストゾーンID**     | 作成後に取得     |

### 9.2 DNSレコード設定

| レコード名                       | タイプ | 値          | TTL  | ルーティングポリシー |
| :------------------------------- | :----- | :---------- | :--- | :------------------- |
| `kawausolms.com`                 | A      | Alias → ALB | -    | シンプル             |
| `www.kawausolms.com`             | A      | Alias → ALB | -    | シンプル             |
| `_acm-validation.kawausolms.com` | CNAME  | ACM検証値   | 300  | シンプル             |

**Aliasレコード設定:**
- Alias Target: `kawausolms-prod-alb`
- Evaluate Target Health: Yes

### 9.3 ACM証明書

| パラメータ           | 設定値                              |
| :------------------- | :---------------------------------- |
| **証明書タイプ**     | パブリック証明書                    |
| **ドメイン名**       | `*.kawausolms.com` (ワイルドカード) |
| **追加ドメイン名**   | `kawausolms.com` (Apex)             |
| **検証方法**         | DNS検証                             |
| **キーアルゴリズム** | RSA 2048                            |
| **自動更新**         | 有効                                |

**検証レコード:**
- Route 53に自動作成
- CNAME レコードで検証

---

## 10. CloudWatch監視・アラート

### 10.1 CloudWatch Logs設定

#### ロググループ

| ロググループ名                                              | 保持期間 | 暗号化     |
| :---------------------------------------------------------- | :------- | :--------- |
| `/aws/kawausolms/prod/application`                          | 7日      | 有効 (KMS) |
| `/aws/kawausolms/prod/system`                               | 7日      | 有効 (KMS) |
| `/aws/rds/cluster/kawausolms-prod-aurora-cluster/error`     | 7日      | 有効 (KMS) |
| `/aws/rds/cluster/kawausolms-prod-aurora-cluster/slowquery` | 7日      | 有効 (KMS) |

**注:** 7日後にS3へエクスポート

### 10.2 CloudWatch Alarms設定

#### EC2 CPU使用率アラーム (Warning)

| パラメータ         | 設定値                                           |
| :----------------- | :----------------------------------------------- |
| **アラーム名**     | `kawausolms-prod-ec2-cpu-warning`                |
| **メトリクス**     | CPUUtilization                                   |
| **ネームスペース** | AWS/EC2                                          |
| **ディメンション** | AutoScalingGroupName = `kawausolms-prod-asg-ec2` |
| **統計**           | Average                                          |
| **期間**           | 300秒 (5分)                                      |
| **閾値**           | >= 70%                                           |
| **評価期間**       | 2回連続                                          |
| **アクション**     | SNS通知 → `kawausolms-prod-sns-alerts`           |

#### EC2 CPU使用率アラーム (Critical)

| パラメータ     | 設定値                                 |
| :------------- | :------------------------------------- |
| **アラーム名** | `kawausolms-prod-ec2-cpu-critical`     |
| **メトリクス** | CPUUtilization                         |
| **閾値**       | >= 85%                                 |
| **評価期間**   | 2回連続                                |
| **アクション** | SNS通知 → `kawausolms-prod-sns-alerts` |

#### EC2 メモリ使用率アラーム (Warning)

| パラメータ         | 設定値                                 |
| :----------------- | :------------------------------------- |
| **アラーム名**     | `kawausolms-prod-ec2-memory-warning`   |
| **メトリクス**     | mem_used_percent (カスタムメトリクス)  |
| **ネームスペース** | CWAgent                                |
| **閾値**           | >= 80%                                 |
| **評価期間**       | 2回連続                                |
| **アクション**     | SNS通知 → `kawausolms-prod-sns-alerts` |

**注:** CloudWatch Agentのインストールが必要

#### RDS CPU使用率アラーム

| パラメータ         | 設定値                                                 |
| :----------------- | :----------------------------------------------------- |
| **アラーム名**     | `kawausolms-prod-rds-cpu-warning`                      |
| **メトリクス**     | CPUUtilization                                         |
| **ネームスペース** | AWS/RDS                                                |
| **ディメンション** | DBClusterIdentifier = `kawausolms-prod-aurora-cluster` |
| **閾値**           | >= 70%                                                 |
| **評価期間**       | 2回連続                                                |
| **アクション**     | SNS通知 → `kawausolms-prod-sns-alerts`                 |

#### RDS 接続数アラーム

| パラメータ         | 設定値                                    |
| :----------------- | :---------------------------------------- |
| **アラーム名**     | `kawausolms-prod-rds-connections-warning` |
| **メトリクス**     | DatabaseConnections                       |
| **ネームスペース** | AWS/RDS                                   |
| **閾値**           | >= 320 (最大400の80%)                     |
| **評価期間**       | 2回連続                                   |
| **アクション**     | SNS通知 → `kawausolms-prod-sns-alerts`    |

#### ALB ターゲット異常アラーム

| パラメータ         | 設定値                                  |
| :----------------- | :-------------------------------------- |
| **アラーム名**     | `kawausolms-prod-alb-unhealthy-targets` |
| **メトリクス**     | UnHealthyHostCount                      |
| **ネームスペース** | AWS/ApplicationELB                      |
| **ディメンション** | TargetGroup = `kawausolms-prod-tg-ec2`  |
| **閾値**           | >= 1                                    |
| **評価期間**       | 2回連続                                 |
| **アクション**     | SNS通知 → `kawausolms-prod-sns-alerts`  |

### 10.3 SNS Topic設定

| パラメータ         | 設定値                          |
| :----------------- | :------------------------------ |
| **Topic名**        | `kawausolms-prod-sns-alerts`    |
| **表示名**         | KawausoLMS Production Alerts    |
| **プロトコル**     | Email                           |
| **エンドポイント** | 運用時に設定 (運用担当者メール) |
| **暗号化**         | 有効 (KMS)                      |

**サブスクリプション確認:**
- メール送信後、確認リンクをクリック

### 10.4 CloudWatch Agent設定 (EC2)

**インストール方法:** Systems Manager Run Command

**設定ファイル (JSON):**

```json
{
  "metrics": {
    "namespace": "CWAgent",
    "metrics_collected": {
      "mem": {
        "measurement": [
          {
            "name": "mem_used_percent",
            "rename": "MemoryUtilization",
            "unit": "Percent"
          }
        ],
        "metrics_collection_interval": 60
      },
      "disk": {
        "measurement": [
          {
            "name": "used_percent",
            "rename": "DiskUtilization",
            "unit": "Percent"
          }
        ],
        "metrics_collection_interval": 60,
        "resources": [
          "/"
        ]
      }
    }
  }
}
```

---

## 11. WAF設定

### 11.1 Web ACL設定

| パラメータ                | 設定値                    |
| :------------------------ | :------------------------ |
| **Web ACL名**             | `kawausolms-prod-waf-acl` |
| **リソースタイプ**        | Regional (ALB)            |
| **関連付けリソース**      | `kawausolms-prod-alb`     |
| **デフォルトアクション**  | Allow                     |
| **CloudWatch メトリクス** | 有効                      |

### 11.2 WAFルール設定

#### ルール1: AWSマネージドルール - Core Rule Set

| パラメータ     | 設定値                                |
| :------------- | :------------------------------------ |
| **ルール名**   | `AWSManagedRulesCommonRuleSet`        |
| **優先度**     | 1                                     |
| **アクション** | Block                                 |
| **説明**       | 一般的なWebアプリケーション脆弱性対策 |

#### ルール2: AWSマネージドルール - Known Bad Inputs

| パラメータ     | 設定値                                 |
| :------------- | :------------------------------------- |
| **ルール名**   | `AWSManagedRulesKnownBadInputsRuleSet` |
| **優先度**     | 2                                      |
| **アクション** | Block                                  |
| **説明**       | 既知の不正入力パターン対策             |

#### ルール3: AWSマネージドルール - SQL Injection

| パラメータ     | 設定値                       |
| :------------- | :--------------------------- |
| **ルール名**   | `AWSManagedRulesSQLiRuleSet` |
| **優先度**     | 3                            |
| **アクション** | Block                        |
| **説明**       | SQLインジェクション攻撃対策  |

#### ルール4: レート制限ルール

| パラメータ       | 設定値                         |
| :--------------- | :----------------------------- |
| **ルール名**     | `RateLimitRule`                |
| **優先度**       | 4                              |
| **ルールタイプ** | Rate-based rule                |
| **レート制限**   | 100リクエスト/5分/IP           |
| **アクション**   | Block (5分間)                  |
| **説明**         | DDoS・ブルートフォース攻撃対策 |

#### ルール5: 地理的制限ルール

| パラメータ       | 設定値                   |
| :--------------- | :----------------------- |
| **ルール名**     | `GeoBlockRule`           |
| **優先度**       | 5                        |
| **ルールタイプ** | Geo match                |
| **許可国**       | JP (日本)                |
| **アクション**   | Block (日本以外)         |
| **説明**         | 日本国内のみアクセス許可 |

### 11.3 WAFログ設定

| パラメータ       | 設定値                         |
| :--------------- | :----------------------------- |
| **ログ記録**     | 有効                           |
| **送信先**       | CloudWatch Logs                |
| **ロググループ** | `aws-waf-logs-kawausolms-prod` |
| **保持期間**     | 30日                           |
| **ログ対象**     | ブロックされたリクエストのみ   |

---

## 12. バックアップ設定

### 12.1 RDS自動バックアップ

| パラメータ                     | 設定値                          |
| :----------------------------- | :------------------------------ |
| **自動バックアップ**           | 有効                            |
| **バックアップ保持期間**       | 7日間                           |
| **バックアップウィンドウ**     | 02:00-03:00 JST                 |
| **ポイントインタイムリカバリ** | 有効 (5分以内の任意時点)        |
| **バックアップ先**             | 同一リージョン (ap-northeast-1) |
| **暗号化**                     | 有効 (KMS)                      |

### 12.2 EC2 AMIバックアップ

#### AWS Backup設定

| パラメータ                 | 設定値                            |
| :------------------------- | :-------------------------------- |
| **バックアッププラン名**   | `kawausolms-prod-ec2-backup-plan` |
| **バックアップ頻度**       | 週次 (日曜日 01:00 JST)           |
| **バックアップウィンドウ** | 01:00-02:00 JST                   |
| **保持期間**               | 4週間 (4世代)                     |
| **ライフサイクル**         | 4週間後に削除                     |
| **バックアップボールト**   | `kawausolms-prod-backup-vault`    |

#### バックアップ対象リソース

| リソースタイプ  | タグ条件      |
| :-------------- | :------------ |
| EC2インスタンス | `Backup=true` |

**注:** Launch Templateに `Backup=true` タグを追加

### 12.3 S3バージョニング (教材データ)

| パラメータ         | 設定値                                  |
| :----------------- | :-------------------------------------- |
| **バケット**       | `kawausolms-prod-contents-{account-id}` |
| **バージョニング** | 有効                                    |
| **MFA削除**        | 無効                                    |
| **ライフサイクル** | 非現行バージョンを即時削除              |

**復元方法:**
- 誤削除時: バージョン履歴から復元
- 保持期間: 最新バージョンのみ

### 12.4 災害復旧 (DR) 設定

| 項目                   | 設定値                      |
| :--------------------- | :-------------------------- |
| **RTO (目標復旧時間)** | 12時間以内                  |
| **RPO (目標復旧時点)** | 24時間以内                  |
| **マルチAZ構成**       | 有効 (自動フェイルオーバー) |
| **AZ障害時の復旧時間** | 数分 (自動)                 |
| **リージョン障害対策** | 現時点では未実装            |

### 12.5 バックアップ通知設定

| パラメータ       | 設定値                                  |
| :--------------- | :-------------------------------------- |
| **通知先**       | SNS Topic: `kawausolms-prod-sns-alerts` |
| **通知イベント** | バックアップ成功・失敗                  |
| **通知方法**     | Email                                   |

---

## 13. コスト管理

### 13.1 予算設定 (AWS Budgets)

| パラメータ     | 設定値                           |
| :------------- | :------------------------------- |
| **予算名**     | `kawausolms-prod-monthly-budget` |
| **予算タイプ** | コスト予算                       |
| **期間**       | 月次                             |
| **予算額**     | ¥200,000                         |
| **開始日**     | 本番稼働月の1日                  |

### 13.2 予算アラート設定

| アラート名         | 閾値 | 金額     | 通知先                      |
| :----------------- | :--- | :------- | :-------------------------- |
| `50%到達アラート`  | 50%  | ¥100,000 | Email (運用担当者)          |
| `75%到達アラート`  | 75%  | ¥150,000 | Email (運用担当者)          |
| `90%到達アラート`  | 90%  | ¥180,000 | Email (運用担当者 + 管理者) |
| `100%到達アラート` | 100% | ¥200,000 | Email (運用担当者 + 管理者) |
| `110%超過アラート` | 110% | ¥220,000 | Email (緊急連絡先)          |

### 13.3 コスト配分タグ

| タグキー      | タグ値       | 用途             |
| :------------ | :----------- | :--------------- |
| `Project`     | `kawausolms` | プロジェクト識別 |
| `Environment` | `prod`       | 環境識別         |
| `CostCenter`  | `elearning`  | コストセンター   |

**注:** 全リソースに適用

### 13.4 コスト最適化計画

| 項目                              | 実施時期 | 期待効果             |
| :-------------------------------- | :------- | :------------------- |
| **リザーブドインスタンス (EC2)**  | 6ヶ月後  | 30〜50%削減          |
| **リザーブドインスタンス (RDS)**  | 6ヶ月後  | 30〜50%削減          |
| **Savings Plans**                 | 6ヶ月後  | 柔軟なコスト削減     |
| **S3ライフサイクル最適化**        | 3ヶ月後  | ストレージコスト削減 |
| **CloudWatch Logs保持期間見直し** | 3ヶ月後  | ログコスト削減       |

---

## 14. セキュリティ設定

### 14.1 AWS Systems Manager (SSM)

#### Session Manager設定

| パラメータ                 | 設定値                     |
| :------------------------- | :------------------------- |
| **有効化**                 | 有効                       |
| **ログ記録**               | CloudWatch Logs            |
| **ロググループ**           | `/aws/ssm/kawausolms-prod` |
| **暗号化**                 | 有効 (KMS)                 |
| **セッションタイムアウト** | 60分                       |

#### Patch Manager設定

| パラメータ                 | 設定値                      |
| :------------------------- | :-------------------------- |
| **パッチベースライン**     | Amazon Linux 2023デフォルト |
| **パッチ適用頻度**         | 月次 (第2土曜日 02:00 JST)  |
| **メンテナンスウィンドウ** | 02:00-04:00 JST             |
| **再起動**                 | 必要に応じて自動再起動      |

### 14.2 AWS Secrets Manager

#### シークレット設定

| シークレット名                       | 説明                  | 自動ローテーション |
| :----------------------------------- | :-------------------- | :----------------- |
| `kawausolms-prod-db-master-password` | RDSマスターパスワード | 無効 (手動管理)    |
| `kawausolms-prod-db-app-user`        | アプリ用DBユーザー    | 有効 (90日)        |

**ローテーション設定 (アプリ用):**
- Lambda関数: 自動作成
- ローテーション間隔: 90日
- 即座のローテーション: 無効

### 14.3 AWS KMS

#### カスタマー管理キー

| キーエイリアス              | 用途      | キーローテーション |
| :-------------------------- | :-------- | :----------------- |
| `alias/kawausolms-prod-ebs` | EBS暗号化 | 有効 (年次)        |
| `alias/kawausolms-prod-rds` | RDS暗号化 | 有効 (年次)        |
| `alias/kawausolms-prod-s3`  | S3暗号化  | 有効 (年次)        |

**キーポリシー:**
- 管理者: フルアクセス
- EC2ロール: 暗号化・復号化のみ

---

## 15. 構築チェックリスト

### 15.1 ネットワーク構築

- [ ] VPC作成 (`10.0.0.0/16`)
- [ ] パブリックサブネット作成 (2AZ)
- [ ] プライベートサブネット作成 (App層 2AZ)
- [ ] プライベートサブネット作成 (DB層 2AZ)
- [ ] インターネットゲートウェイ作成・アタッチ
- [ ] NAT Gateway作成 (2AZ、Elastic IP割り当て)
- [ ] ルートテーブル作成・関連付け

### 15.2 セキュリティ設定

- [ ] セキュリティグループ作成 (ALB, EC2, RDS)
- [ ] IAMロール作成 (EC2, RDS監視)
- [ ] IAMポリシー作成・アタッチ
- [ ] KMSキー作成 (EBS, RDS, S3)

### 15.3 データベース構築

- [ ] RDS Auroraクラスター作成
- [ ] Writerインスタンス作成
- [ ] Readerインスタンス作成 (最小1台)
- [ ] Reader Auto Scaling設定
- [ ] パラメータグループ作成・適用
- [ ] Secrets Manager設定 (DB認証情報)
- [ ] RDSログ設定 (エラー、スロークエリ)

### 15.4 コンピューティング構築

- [ ] Launch Template作成
- [ ] Auto Scaling Group作成
- [ ] スケーリングポリシー設定
- [ ] CloudWatch Agent設定

### 15.5 ロードバランサー構築

- [ ] ALB作成
- [ ] ターゲットグループ作成
- [ ] HTTPSリスナー作成
- [ ] ヘルスチェック設定
- [ ] ALBアクセスログ設定

### 15.6 ストレージ構築

- [ ] S3バケット作成 (教材、ログ、ALBログ)
- [ ] バケットポリシー設定
- [ ] ライフサイクルポリシー設定
- [ ] バージョニング設定 (教材バケット)

### 15.7 DNS・証明書設定

- [ ] Route 53ホストゾーン作成
- [ ] ACM証明書リクエスト
- [ ] DNS検証レコード追加
- [ ] Aレコード作成 (ALB Alias)

### 15.8 WAF設定

- [ ] Web ACL作成
- [ ] マネージドルール追加
- [ ] レート制限ルール追加
- [ ] 地理的制限ルール追加
- [ ] ALBに関連付け

### 15.9 監視・アラート設定

- [ ] CloudWatch Logsグループ作成
- [ ] CloudWatch Alarms作成 (CPU, メモリ, RDS)
- [ ] SNS Topic作成
- [ ] Email サブスクリプション設定・確認

### 15.10 バックアップ設定

- [ ] RDS自動バックアップ有効化
- [ ] AWS Backupプラン作成 (EC2 AMI)
- [ ] バックアップボールト作成
- [ ] S3バージョニング有効化

### 15.11 コスト管理設定

- [ ] AWS Budgets作成
- [ ] 予算アラート設定
- [ ] コスト配分タグ適用

### 15.12 セキュリティ追加設定

- [ ] Systems Manager Session Manager有効化
- [ ] Patch Manager設定
- [ ] Secrets Manager設定
- [ ] KMSキーローテーション有効化

---

## 16. 構築後の確認事項

### 16.1 接続確認

- [ ] ALB経由でEC2へのHTTPSアクセス確認
- [ ] EC2からRDSへのDB接続確認
- [ ] EC2からS3へのアクセス確認
- [ ] Systems Manager経由でEC2へのアクセス確認

### 16.2 動作確認

- [ ] Auto Scalingのスケールアウト動作確認
- [ ] Auto Scalingのスケールイン動作確認
- [ ] ALBヘルスチェック動作確認
- [ ] RDS Reader Auto Scaling動作確認

### 16.3 監視確認

- [ ] CloudWatch Logsへのログ出力確認
- [ ] CloudWatch Alarmsの動作確認
- [ ] SNS通知の受信確認
- [ ] WAFログの記録確認

### 16.4 バックアップ確認

- [ ] RDS自動バックアップの実行確認
- [ ] EC2 AMIバックアップの実行確認
- [ ] S3バージョニングの動作確認

### 16.5 セキュリティ確認

- [ ] セキュリティグループルールの確認
- [ ] IAMロール権限の最小化確認
- [ ] 暗号化設定の確認 (EBS, RDS, S3)
- [ ] WAFルールの動作確認

---

## 付録A: 運用時に設定が必要な項目

| 項目                  | 設定内容                   | 担当       |
| :-------------------- | :------------------------- | :--------- |
| **SSH接続元IP**       | 開発会社の固定IPアドレス   | 開発会社   |
| **SNS通知先メール**   | 運用担当者のメールアドレス | 運用チーム |
| **Secrets Manager**   | RDSマスターパスワード      | 構築チーム |
| **Secrets Manager**   | アプリ用DBユーザー認証情報 | 構築チーム |
| **CloudWatch Alarms** | 通知先メールアドレス       | 運用チーム |
| **AWS Budgets**       | 予算アラート通知先         | 管理者     |

---

## 付録B: 参考情報

### B.1 想定月額コスト (概算)

| サービス                 | 構成                       | 月額コスト (概算)      |
| :----------------------- | :------------------------- | :--------------------- |
| EC2 (t3.medium)          | 2〜10台 (平均4台)          | ¥40,000〜¥50,000       |
| RDS Aurora (db.t3.large) | Writer 1台 + Reader 1〜3台 | ¥48,000〜¥64,000       |
| ALB                      | 1台                        | ¥3,000〜¥5,000         |
| NAT Gateway              | 2台                        | ¥10,000〜¥15,000       |
| S3                       | 教材5GB + ログ             | ¥500〜¥1,000           |
| Route 53                 | ホストゾーン + クエリ      | ¥500〜¥1,000           |
| CloudWatch               | ログ + メトリクス          | ¥5,000〜¥10,000        |
| WAF                      | Web ACL + ルール           | ¥1,200〜¥2,000         |
| データ転送               | 1,800人対応                | ¥10,000〜¥20,000       |
| **合計**                 | -                          | **¥118,200〜¥168,000** |

**注:** 実際のコストは利用状況により変動

### B.2 負荷試算サマリー

| 項目                     | 値            |
| :----------------------- | :------------ |
| ピーク時同時接続ユーザー | 1,800人       |
| 総クエリ数 (ピーク時)    | 300 QPS       |
| 読み取りクエリ           | 240 QPS (80%) |
| 書き込みクエリ           | 60 QPS (20%)  |
| EC2台数 (ピーク時)       | 4〜6台        |
| RDS Reader数 (ピーク時)  | 2〜3台        |

---

**ドキュメント作成日:** 2025年11月22日
**バージョン:** 2.0
**作成者:** AWS構築チーム
**承認者:** プロジェクトマネージャー
