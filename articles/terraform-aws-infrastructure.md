---
title: "Terraformでdev/prodを分離したAWS Webアプリ基盤を構築してみた"
emoji: "🏗️"
type: "tech"
topics: ["aws", "terraform"]
published: true
---

## はじめに

ポートフォリオ「Dev-Portfolio」として、Terraformを用いてAWS上に社内向け申請管理システム基盤を構築しました。
FastAPIを用いたREST APIを、ALB / EC2 Auto Scaling / RDS PostgreSQLを中心としたWeb三層構成で動かしています。
冗長化・可用性、セキュリティ、運用性、コストを考慮して設計しました。

制作は、要件定義 → 基本設計 → AWSマネジメントコンソールでの構築 → TerraformによるIaC化の順に進めました。
コンソール上で各リソースの役割と通信経路、依存関係を確認し、その理解をTerraformの変数やmoduleの参照関係へ落とし込んでいます。

本記事では、TerraformによるAWS基盤の構築と、共通構成を保ちながらdev/prodの要件差をコード化した方法を中心に紹介します。
ソースコード・Terraform・要件定義書・基本設計書・README・構成図は、[Dev-PortfolioのGitHubリポジトリ](https://github.com/sakuyaxx21-sys/dev-portfolio)で公開しています。

## 構築したシステムと要件

アプリケーションには、一般ユーザーが申請を作成し、管理者が承認・却下する機能を実装しています。
インフラについては、`docs/requirements.md`で次の方針を定めています。

<!-- markdownlint-disable MD033 -->

| 観点 | 要件・方針 |
| --- | --- |
| 通信経路 | インターネットからの入口をALBに限定し、EC2とRDSを直接公開しない |
| 可用性 | 複数AZにまたがる構成とし、ALBとASGを利用する<br>prodではRDS Multi-AZを有効にする |
| 運用接続 | EC2へのSSH接続は行わず、Systems Managerを利用する |
| 機密情報 | DB認証情報とアプリケーションのSecretをSecrets Managerで管理する |
| 運用 | インフラをTerraformで管理し、ログ収集・監視通知も構成に含める |
| コスト | devは検証コストを抑え、prodは本番を想定した可用性と保護を重視する |

<!-- markdownlint-enable MD033 -->

実行基盤にはEC2＋Dockerを選びました。
ECSで抽象化する前に基盤を理解することと、Dockerでポータビリティを持たせることを意図しています。

## AWSアーキテクチャ

東京リージョンの`ap-northeast-1a`と`ap-northeast-1c`に、Public、Private App、Private DBの3種類のサブネットをそれぞれ配置します。

![Dev-PortfolioのAWSアーキテクチャ全体図](/images/terraform-aws-infrastructure/architecture.png)

図は2AZを利用する基盤全体を示しています。
NAT GatewayとRDSの構成はprodを想定したもので、devとの差分は後述します。
ASGは両AZのPrivate App Subnetを対象とし、コストを考慮して希望台数を1台、最大台数を2台にしています。

Route 53は名前解決を担い、HTTPリクエストは利用者からALBへ届きます。
ALBの80番ポートはHTTPSへリダイレクトし、443番ポートでTLSを終端します。
ALBからEC2への転送はHTTPです。

Route 53の公開ホストゾーンは`data "aws_route53_zone"`で既存のものを参照しています。
Terraformで作成するのは、アプリケーション用のAliasレコードやACMのDNS検証レコードなどです。

サブネットの役割はルートテーブルでも分けています。

| サブネット | 配置するリソース | インターネット向けデフォルトルート |
| --- | --- | --- |
| Public | ALB、NAT Gateway | Internet Gateway |
| Private App | EC2 Auto Scaling Group | NAT Gateway |
| Private DB | RDS PostgreSQL | なし |

EC2では起動時にパッケージやDockerイメージを取得し、AWS APIにもアクセスします。
Private Appからの外向き通信はNAT Gatewayを経由させ、DB用サブネットとは経路を分けています。

## Terraformを採用した理由

Terraformを採用したのは、コンソールで構築した基盤をコードとして管理し、同じ設計をもとに環境ごとの要件を反映できるようにするためです。
インフラ構成の属人化を防ぎ、構成と変更内容をコードから追える状態にすることを重視しました。

今回のIaC化では、次の3点を軸に整理しています。

- **共通構成はmoduleにまとめる**：VPCやALB、RDSの定義をdev/prodから再利用します。
- **環境差分は変数で渡す**：NAT Gateway数やRDS Multi-AZなど、要件によって変わる値を環境側で管理します。
- **依存関係は参照でつなぐ**：Subnet IDやSecurity Group IDをoutputで受け渡し、リソース同士の関係を明示します。

## Terraformのディレクトリ構成

主要な構成は次のとおりです。
各ディレクトリ内のファイルは一部省略しています。

```text
infra/
├── bootstrap/             # 環境のstateを保存するS3バケット
├── envs/
│   ├── dev/               # dev用のroot module
│   └── prod/              # prod用のroot module
├── modules/
│   ├── network/           # VPC、Subnet、ルート、NAT、Security Group
│   ├── app/               # ALB、ACM、DNSレコード、Launch Template、ASG
│   ├── db/                # DB Subnet Group、RDS
│   ├── security/          # IAM、KMS、Secrets Manager、WAF
│   ├── operations/        # ログ保存先、SNS、Slack通知連携
│   └── monitoring/        # CloudWatch Alarm
├── .terraform-version
└── README.md
```

`envs`で環境ごとの値を受け取り、共通の`modules`を組み合わせています。
`dev/main.tf`と`prod/main.tf`のmodule呼び出しは同じで、主な差分は`variables.tf`の既定値と`backend.tf`のstate保存先キーです。

`variables.tf`はmoduleへの入力、`outputs.tf`は作成したリソースのIDやARNなどの受け渡しを定義します。
`locals.tf`では環境名とプロジェクト名から名前の接頭辞を作り、Providerの`default_tags`で`Project`、`Env`、`ManagedBy`を設定しています。

リポジトリの`.terraform-version`はTerraform `1.15.5`です。
環境側の`versions.tf`ではAWS Providerを`>= 5.61.0, < 6.0.0`に制約し、dev/prodのロックファイルでは`5.100.0`を選択しています。

### bootstrapでstateの保存先を分ける

`bootstrap`では、環境のstateを保存するS3バケットを作成します。
バージョニング、パブリックアクセスのブロック、SSE-S3による暗号化を設定しています。

dev/prodは同じバケットを参照し、キーをそれぞれ`envs/dev/terraform.tfstate`と`envs/prod/terraform.tfstate`に分けています。
環境ディレクトリごとにS3 Backendを持つ構成で、Terraform workspaceによる切り替えではありません。

`backend.tf`では`encrypt = true`と`use_lockfile = true`を設定しています。
S3 Backendのstate lockを使う構成です。
設定の詳細は[HashiCorpのS3 Backendドキュメント](https://developer.hashicorp.com/terraform/language/backend/s3)を参照してください。

この分離により、stateの保存先を用意する構成と、アプリケーション基盤を作る構成を別のroot moduleとして扱っています。

## dev/prodで変えていること

要件定義では、devは低コストで検証しやすく、prodは本番を想定した可用性と保護を重視する方針です。
現在の`infra/envs/{dev,prod}/variables.tf`にある既定値は次のとおりです。

| 設定 | dev | prod |
| --- | --- | --- |
| NAT Gateway数 | 1 | 2 |
| RDS Multi-AZ | 無効 | 有効 |
| RDS自動バックアップ保持 | 0日 | 7日 |
| RDS削除保護 | 無効 | 有効 |
| アプリ用Secretの削除復旧期間 | 0日 | 30日 |
| KMSキー削除の待機期間 | 7日 | 30日 |
| ALBログ用S3の`force_destroy` | `true` | `false` |

EC2は両環境とも`t3.micro`、RDSは`db.t4g.micro`を既定値にしています。
インスタンスサイズを変えるよりも、NAT Gatewayの配置、DBの冗長化、データ保護に環境差分を持たせた構成です。

prodではRDS Multi-AZと7日間の自動バックアップ、削除保護を有効にしています。
障害に備える設定と、データの復旧・誤削除に備える設定を、それぞれ変数として表現しています。
一方、devでは検証環境としてのコストと作り直しやすさを優先しています。

## 主要moduleをどう接続したか

### network：NATの台数とルートを一緒に切り替える

NAT Gatewayを1台にする場合と2台にする場合では、作成数だけでなく、Private App Subnetの接続先も変わります。

`infra/modules/network/network.tf`では、次のようにルートを定義しています。

```hcl
resource "aws_route" "private_app_default" {
  count = length(aws_route_table.private_app)

  route_table_id         = aws_route_table.private_app[count.index].id
  destination_cidr_block = "0.0.0.0/0"
  # Reuse the last NAT gateway when fewer NATs than app subnets are requested.
  nat_gateway_id = aws_nat_gateway.main[min(count.index, var.nat_gateway_count - 1)].id
}
```

現在の2AZ構成では、devの両サブネットは同じNAT Gatewayを参照します。
prodはサブネットとNAT Gatewayの配列順が対応し、それぞれ同一AZのNAT Gatewayを参照します。

devではNAT Gatewayを1台に抑え、固定費を削減しています。
その分、両AZの外向き通信は同じNAT Gatewayに依存します。
prodでは各AZにNAT Gatewayを配置し、片方のAZのNATに両方の通信経路を集約しない設計です。
このように、台数とルートの参照先を一緒に切り替えることで、コストと可用性の方針をコードへ反映しています。

### module間はoutputを通して接続する

`network`で作成したPrivate DB SubnetとSecurity Groupを、環境側の`main.tf`で`db`へ渡します。
`infra/envs/dev/main.tf`から必要な部分を抜粋します。

```hcl
module "db" {
  source = "../../modules/db"

  name_prefix           = local.name_prefix
  private_db_subnet_ids = module.network.private_db_subnet_ids
  db_security_group_id  = module.network.db_security_group_id

  # 省略
  db_multi_az          = var.db_multi_az

  backup_retention_period = var.backup_retention_period
  deletion_protection     = var.deletion_protection
  skip_final_snapshot     = var.skip_final_snapshot

  kms_key_arn = module.security.kms_key_arn
}
```

リソースIDを手で転記せず、`module.network`や`module.security`のoutputを参照しています。
環境側で「どのネットワークに、どの暗号鍵と可用性設定でDBを作るか」が読み取れます。

同様に、DBの接続先や管理対象SecretのARNは`app`へ渡し、EC2の起動設定に利用します。

### app：EC2を起動する手順もコードに含める

`app`ではALBだけでなく、Launch TemplateとASGも定義しています。
Launch Templateのuser dataは、`user_data.sh.tftpl`へ変数を渡して生成します。

起動時には、DockerやCloudWatch Agentなどを準備し、Secrets Managerから認証情報を取得して環境変数ファイルを生成します。
その後、Dockerイメージを取得し、Alembicのマイグレーションとアプリケーション起動を実行します。

ASGの定義から、配置とヘルスチェックに関係する部分を抜粋します。

```hcl
resource "aws_autoscaling_group" "app" {
  name                = "${local.name_prefix}-asg-app"
  min_size            = var.asg_min_size
  max_size            = var.asg_max_size
  desired_capacity    = var.asg_desired_capacity
  vpc_zone_identifier = var.private_app_subnet_ids
  target_group_arns   = [aws_lb_target_group.app.arn]
  health_check_type   = "ELB"

  launch_template {
    id      = aws_launch_template.app.id
    version = aws_launch_template.app.latest_version
  }

  # 省略
}
```

出典は`infra/modules/app/app.tf`です。
ALBのTarget GroupをASGに関連付け、ELBのヘルスチェックを利用しています。
異常と判定されたインスタンスをASGが置き換える仕組みについては、[AWSのヘルスチェックの説明](https://docs.aws.amazon.com/autoscaling/ec2/userguide/health-checks-overview.html)に沿った設定です。

ASGで希望台数を維持し、複数AZのPrivate App Subnetを対象として最大2台まで稼働できる構成にしています。

EC2の起動設定までコードに含めることで、置き換え時にも同じ手順でアプリケーションを起動する構成にしています。
起動処理はイメージ取得やDB接続も伴うため、確認時にはcloud-initのログとTarget Groupの状態を合わせて見ることが必要です。

## セキュリティをコードで表現する

### Security Groupは通信元の役割で制限する

EC2のSecurity GroupはALBのSecurity Groupからのアプリケーションポートだけを受け付けます。
RDS側も同様に、EC2のSecurity Groupを通信元に指定しています。

`infra/modules/network/security_group.tf`のDB側の抜粋です。

```hcl
resource "aws_security_group" "db" {
  # 省略
  ingress {
    description     = "Allow PostgreSQL from app servers"
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
  }
  # 省略
}
```

特定のEC2のIPアドレスではなくSecurity Groupを参照するため、ASGでインスタンスが入れ替わる構成にも対応できます。
EC2の22番ポートを許可するルールはありません。
EC2のIAM Roleには`AmazonSSMManagedInstanceCore`を付与しています。

### DBの非公開化と認証情報管理を分けて設定する

`infra/modules/db/db.tf`では、Private DB Subnetへの配置に加え、公開アクセスを無効にし、ストレージを暗号化しています。

DBのマスターパスワードはRDSによるSecrets Manager管理を有効にしています。
EC2にはそのSecretを読む権限を与え、起動時に取得します。
アプリケーション側のSecretは別途`security`で生成・保存しています。

ここで、Secrets Managerを使うことと、Terraformのstateに機密値が残らないことは別です。
現在のアプリ用Secretは`random_password`と`aws_secretsmanager_secret_version`で管理しており、stateも機密情報として保護する必要があります。
`sensitive`は通常のCLI出力などで値の表示を抑制する指定であり、stateへの保存自体を防ぐものではありません。
[HashiCorpの機密データ管理の説明](https://developer.hashicorp.com/terraform/language/manage-sensitive-data)で、この違いを確認できます。

## 基盤の運用もTerraformの管理対象にする

`operations`ではログ保存先と通知先を、`monitoring`ではASG、ALB、Target Group、RDSのAlarmを定義しています。
アプリケーションを動かすリソースに加えて、起動後の状態を確認するための構成もmoduleとして管理しています。

CIには`terraform fmt -check`と、dev/prodそれぞれの`terraform validate`を組み込んでいます。
共通moduleの変更を両環境から確認することで、環境側との入力・出力の不整合を検出しやすくしています。

GitHub ActionsとSSMによるデプロイ、重要度別の監視通知、WAFのルール調整については、別の記事で詳しく扱う予定です。

## GUI構築からIaC化して得た理解

コンソールで構築した内容をTerraformに落とし込む過程では、画面ごとに設定していた項目を、リソース間の関係として整理する必要がありました。
特に意識したのは、通信経路、環境差分、起動処理の3点です。

### 通信経路を設定の組み合わせとして捉える

EC2をPrivate Subnetへ配置するだけで、必要な通信がすべて整うわけではありません。
ALBから受け付ける通信はSecurity Groupで制限し、イメージ取得などの外向き通信はNAT Gatewayへのルートで実現します。
DBには外向きのデフォルトルートを持たせず、EC2からの接続をSecurity Groupで許可します。
この関係をコードで追うことで、配置・経路・通信許可を分けて説明できるようになります。

### 環境差分を設計方針と対応させる

共通moduleにする際には、何を共通にして何を環境側へ渡すかを整理しました。
サブネットの役割やレイヤー間の接続は共通とし、NAT Gateway数やRDS Multi-AZ、バックアップ保持期間を環境ごとの値として扱っています。
これにより、devとprodの違いを個別の設定の集まりではなく、コスト・可用性・データ保護の方針として説明できます。

### リソースの依存関係を起動処理までつなぐ

DB moduleのoutputは、EC2の起動時に利用する接続先にもつながります。
Terraformで作るAWSリソースと、その上で動くアプリケーションを切り離さず、認証情報の取得、マイグレーション、コンテナ起動まで一連の構成として整理しました。
moduleを分割しても、入力と出力をたどることで全体のつながりを把握できます。

## まとめ

Dev-Portfolioでは、要件定義・基本設計からAWS上での構築を経て、基盤をTerraformでIaC化しました。
`bootstrap`、`envs`、`modules`に構成を分け、通信経路とリソース間の依存関係を参照でつなぎ、dev/prodの違いを変数で表現しています。

NAT Gateway、RDS Multi-AZ、バックアップや削除保護を通じて、コストと可用性・データ保護の違いを設計へ反映しました。
AWSの構成を理解したうえでコードに落とし込むことが、再利用でき、意図を説明できるインフラにつながると感じています。
