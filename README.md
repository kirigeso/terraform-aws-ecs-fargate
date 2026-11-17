# Terraform × AWS ECS Fargateポートフォリオ

未経験からSREを目指すために作成した、AWS ECS (Fargate)のインフラ構築リポジトリです。    
チュートリアルをなぞるだけではなく、「現場で使えるレベルとは何か？」を考え、    
Stateの排他制御、CI/CDの完全自動化、tfsecとOIDCによるセキュリティにこだわって実装しました。

---

## アーキテクチャ
<img width="1435" height="613" alt="image" src="https://github.com/user-attachments/assets/22bab105-834e-4391-8342-3ed3887a1367" />

---

## 技術スタック

| カテゴリ | 詳細 |
| :--- | :--- |
| IaC | Terraform v1.14.2 (Module設計) |
| Compute | Amazon ECS (Fargate) |
| Container Registry | Amazon ECR |
| Network | VPC, Public/Private Subnet, ALB, VPC Endpoints |
| CI/CD | GitHub Actions (GitOps) |
| Security | IAM (Least Privilege), OIDC, tfsec |
| Observability | AWS X-Ray, OpenTelemetry(ADOT), CloudWatch (Logs, Metrics, Alarm), SNS |
| State Management | Amazon S3 + DynamoDB (State Lock) |

---

## 想定コスト試算
個人開発レベルのトラフィックを想定し、セキュリティとコストのバランスを最適化した構成の月額の試算です。
NAT Gateway（約$30/月）を廃止し、VPCエンドポイントを使うことで固定費を抑えています。

| Service | Spec / Configuration | Cost | Note |
| :--- | :--- | :--- | :--- |
| ALB | Application Load Balancer | $18.00 | 常時稼働 |
| Fargate | 0.25 vCPU / 0.5 GB Mem | $10.00 | 2タスク稼働想定 |
| VPC Endpoints | Interface Type (x3) | $25.00 | ECR(api/dkr), Logs用 |
| CloudWatch | Logs & Metrics | $3.00 | ログ量に依存 |
| Others | S3, DynamoDB, ECR, Data Transfer | $2.00 | 従量課金 |
| Total | | ~$58.00 / month | NAT GW利用時より約30%削減 |

---

## 技術選定の背景

### Compute: なぜEC2ではなくECS Fargateか？
本プロジェクトでは「インフラ管理コストの最小化」を優先しました。
- 運用の面では、EC2インスタンスのOSパッチ適用やスケーリング管理が不要になり、コンテナ定義だけに集中できるため。
- セキュリティの面では、ホストOSへのSSH管理が不要になり、攻撃対象領域を最小化できるため。

### Network: なぜVPCエンドポイントを採用したか？
セキュリティとコストのバランスを考慮し、インターネットを経由しないプライベートな通信経路を確保しました。
- セキュリティの面では、ECRやCloudWatch Logsへの通信をAWS内網（PrivateLink）に閉じることで、セキュリティリスクを軽減しています。
- コストと性能の面では、特にS3への通信（Dockerレイヤー取得など）にゲートウェイ型エンドポイントを採用しました。NAT Gatewayを経由させる場合に比べ、通信コストをゼロにしつつ、スループットを向上させています。

---
## こだわりポイント

### 1. State管理と排他制御
チーム開発を想定し、Terraform StateをS3で一元管理し、DynamoDBによるState Lockを実装しています。    
これにより、複数人が同時に`terraform apply`を実行してステートが破損する事故を、ロックの仕組みで防いでいます。

### 2. GitOpsスタイルのCI/CDパイプライン
GitHub Actionsを使い、インフラとアプリケーションのデプロイを完全自動化しました。
- Infrastructure: PR作成時に`terraform plan`を実行し、結果をBotがPRにコメントします。`main`へのマージで`terraform apply`が自動実行されます。
- Application: コード修正をトリガーにDocker Build → ECR Push → ECS Rolling Updateまでを全自動で行います。

### 3. セキュリティ・ガードレール(Shift Left)
- OIDC認証: AWSアクセスキー（AK/SK）を一切発行せず、GitHub ActionsとAWSをOIDCで連携させることで、認証情報の漏洩リスクを排除しました。
- tfsec導入: CIパイプラインに静的解析ツール`tfsec`を組み込み、セキュリティリスクのある設定（例: S3公開設定漏れなど）をデプロイ前に検知・ブロックする仕組みを構築しました。
- 最小権限の原則: IAMロールにはWildcard (`*`)を極力使用せず、必要なアクションのみを許可する厳格なポリシーにしています。

### 4. 信頼性向上のための自動ロールバック
ECSのDeployment Circuit Breakerを有効化しています。    
デプロイしたコンテナがヘルスチェックに失敗した場合、自動的に健全な旧バージョンへロールバックし、サービスダウンタイムを最小限に抑えます。

### 5. コスト最適化とリソース管理
ストレージのコストが増え続けないよう、ライフサイクルポリシーを導入しました。
- ECR: 過去のイメージが無限に溜まらないよう、最新の10世代のみを保持し、古いイメージを自動削除する設定を入れています。
- S3 (State Bucket): Terraform Stateのバージョニングを有効化しつつ、古すぎるバージョン（非現行バージョン）を自動的に削除するルールを適用し、容量を節約しています。

### 6. 可観測性（Observability）の実装
障害発生時に「システム内のどこでボトルネックが起きているか」を開発者が即座に特定できるよう、分散トレーシング基盤を構築しました。

- OpenTelemetryによる計装とサイドカーパターン: アプリケーション（FastAPI）には特定のベンダーに依存しないOpenTelemetryを組み込み、ADOT（AWS Distro for OpenTelemetry）コンテナをサイドカーとして配置して、トレースデータの変換とAWS X-Rayへの送信をすべて任せました。これにより、開発者はビジネスロジックの実装に専念でき、通信の再送や認証はインフラ側が受け持つ「関心の分離」を実現しています。

- 閉域網でのトレースデータの送信: タスク内のコンテナ間はlocalhost（gRPC）で高速に連携し、外部への送信はVPCエンドポイントを経由させることで、インターネットに出ることなくトレースデータをAWS X-Rayへ届ける構成にしています。
---

## Terraformの設計

Terraformの構成は再利用性と環境分離を意識して、2レイヤーに分割しています。

### `env/` (Environment Layer)
環境固有の値（CIDR, Region, Instance Size）を定義し、モジュールを呼び出します。
> `env`ディレクトリ内では`aws_`リソースを直接定義せず、インフラの「組み立て」に専念させています。

### `modules/` (Resource Layer)
AWSリソースの具体的な定義を行う「部品」です。
各モジュールは単一責任の原則に基づき分割されており、環境（dev/stg/prod）に依存しない設計としています。

---

## ディレクトリ構成
```text
├── env/
│   └── dev/                 # 開発環境用エントリポイント
│       ├── main.tf          # モジュール呼び出し
│       ├── variables.tf     # 環境変数定義
│       └── terraform.tfvars # 具体的なパラメータ
├── modules/                 # 再利用可能なリソース部品
│   ├── vpc/
│   ├── alb/
│   ├── ecs/
│   ├── ecr/
│   ├── iam/
│   ├── sns/
│   ├── s3/
│   ├── dynamodb/
│   └── cloudwatch/
├── .github/
│   └── workflows/
│       ├── docker-ecr.yml      # アプリデプロイ用 (Build & Push)
│       ├── terraform-plan.yml  # PR時のPlan自動実行 & tfsec
│       └── terraform-apply.yml # Merge時のApply自動実行
├── app/
│   ├── Dockerfile
│   └── nginx.conf
```

---

## SLO/SLI

本プロジェクトでは明確な信頼性目標（SLO）を定義して、運用設計を行っています。

| Metric | SLI (Service Level Indicator) | SLO (Service Level Objective) | Monitoring |
| --- | --- | --- | --- |
| Availability | (Total Requests - 5xx Errors) / Total Requests | 99.9% | CloudWatch Alarm (ALB 5xx Count) |
| Latency | Target Response Time | P99 < 1.0s | CloudWatch Alarm (TargetResponseTime) |

### 障害対応フロー
アラート発報から復旧までのフローを、次のように定めています。    
現在は個人開発のためメール通知ですが、チーム運用を想定した設計を意識しています。

1. Detection (検知): CloudWatch AlarmがSLO違反（レイテンシ悪化 / 5xxエラー）を検知します。
2. Notification (通知): SNS経由で管理者へアラートメールを送信します。（今後の予定: AWS Chatbotを導入し、Slackへの通知とChatOpsによる迅速な情報共有を想定）
3. Triage (一次対応): CloudWatch Logs Insightsからエラー原因を特定し、ECSタスクの再起動、または旧バージョンへのロールバックを実施します。
4. Post-Mortem (事後分析): 障害原因を特定し、再発防止策（コード修正やオートスケーリング設定の見直し）を実施します。
---

## 開発中にハマったポイント
### 1. ECSからECRへのPullがタイムアウトする問題
構築中、コンテナが起動せず`ImagePullBackOff`のようなタイムアウトエラーが発生しました。    
セキュリティグループやルートテーブルは正しいはずなのに繋がらない...と数時間悩みました。

【原因と解決】    
ECRのエンドポイント設定（PrivateLink）はしていましたが、「ECRの実体（レイヤーデータ）はS3にある」という仕様を見落としていました。    
Docker PullにはS3への通信経路が必要だったため、ゲートウェイ型のS3エンドポイントを追加することで解決しました。    
「エンドポイントを作れば繋がる」という思い込みが原因でした。AWSの裏側の仕組みを理解する良い経験になりました。

### 2. CI/CD構築におけるIAM権限の「イタチごっこ」と最小権限の設計
Terraformの実行環境をローカル（AdministratorAccess）からGitHub Actions（OIDCロール）へ移行する際、権限不足によるパイプラインの失敗が多発しました。    

【課題：ブートストラップ問題】    
最初は「ECRへのPush」などアプリケーションデプロイに必要な最小限の権限のみを与えていました。    
しかし、Terraform Applyはインフラ全体の変更（IAMロール自体の作成や、CloudWatchアラームの追加など）を伴うため、    
リソースを追加するたびに「CIが権限エラーで落ちる → ポリシーを追加する」というイタチごっこ（手戻り）が発生しました。

【解決と学び】    
「Terraformを実行するCIロール」には、インフラを変更するための広範な権限（PowerUserに近い権限とIAM操作権限）が必要であると再認識しました。    
最終的に、`modules/iam/main.tf`でTerraform Apply用のポリシーを体系的に整理し、    
次のカテゴリごとに必要なアクションを明示的に許可することで、セキュリティと運用効率のバランスを取りました。

- State管理: S3/DynamoDBへのアクセス
- リソース操作: VPC, ALB, ECS, CloudWatchへのフルアクセス
- IAM操作: `iam:PassRole`や`iam:CreateRole`（Terraformがロールを作成・紐付けするために必須）

### 3. ADOTサイドカーの導入で起きた通信の問題
OpenTelemetryとX-Rayを連携させる際、インフラ側の問題が複数重なり、トラブルシューティングの力が問われました。

【課題と仮説検証のプロセス】

1. パブリックイメージの取得タイムアウト: NAT Gatewayを持たないプライベートサブネットのため、public.ecr.awsからADOTのイメージを取得できず、タスクが起動しませんでした。解決策として、手元のPCから自分のプライベートECRへイメージをミラーリングし、VPCエンドポイント（dkr/api）経由で安全にPullする構成に変更しました。

2. 権限と出口（VPCエンドポイント）の欠如: コンテナは起動したものの、X-Rayにトレースが表示されませんでした。CloudWatchのADOTコンテナのログ（一次情報）を調査した結果、2つの原因を特定しました。

    - メトリクス送信用のCloudWatchAgentServerPolicyのアタッチ漏れによる権限エラー。
    - X-Rayサービス自体へ到達するための「X-Ray用VPCエンドポイント」の作成漏れ。

【学び】     
「ログという一次情報を起点にエラーを特定する」「ネットワークの物理的な出口（VPCエンドポイント）と論理的な権限（IAM）を分けて考える」という、インフラとアプリケーションの境界線で起こるトラブルを解決する非常に実践的な経験を得ることができました。

---

## 今後の展望
### 1. Service Auto Scalingの導入
現在は固定タスク数で運用していますが、CPU/メモリ使用率に応じたTarget Tracking Scalingを導入し、    
スパイクアクセスへの耐性と、夜間帯のコスト削減（スケールイン）の両立を目指します。

### 2. Blue/Greenデプロイメント(AWS CodeDeploy)
現在のローリングアップデートに加え、CodeDeployを導入することでBlue/Greenデプロイメントを実現したいと考えています。    
これにより、本番切り替え前の検証が可能になり、リリースリスクをさらに低減させます。

### 3. AWS WAFによるセキュリティ強化
ALBにAWS WAF (Web Application Firewall)をアタッチし、一般的なWeb脆弱性（OWASP Top 10）への防御層を追加することを検討しています。

### 4. 可観測性(Observability)の強化     
X-Rayによる分散トレーシング（Trace）と、CloudWatchによるメトリクス（Metric）、ログ（Log）の基本基盤が整いました。    
今後は「トレースIDをアプリケーションの標準ログにも自動付与する」仕組みを追加し、    
エラー発生時に「X-Rayのグラフから、該当リクエストのCloudWatch Logsへそのまま移って原因を特定できる状態（TraceとLogの相関付け）」を目指します。

---

## 補足

本リポジトリはポートフォリオとしての可読性を重視し、    
インフラ(Terraform)とアプリケーションを単一のリポジトリで管理するMonorepo構成を採用しています。

実際のチーム開発や本番運用においては、ライフサイクルの違いや権限分離の観点から、次のようにリポジトリを分割する構成が一般的であると理解しています。
- Infrastructure Repo: Terraformコードのみを管理（SREチームがオーナー）
- Application Repo: ソースコードとDockerfileを管理（開発チームがオーナー）
