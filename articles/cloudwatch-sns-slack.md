---
title: "CloudWatch Alarm × SNS × Slackで重要度別の監視通知を構築してみた"
emoji: "🔔"
type: "tech"
topics: ["aws", "cloudwatch", "sns", "slack"]
published: true
---

## はじめに

ポートフォリオ「Dev-Portfolio」として、ALB / EC2 Auto Scaling / RDS PostgreSQLを利用した社内向け申請管理システムを構築しています。
今回は、その基盤の状態をCloudWatch Alarmで判定し、重要度に応じてSNSとSlackの通知先を分けた監視設計を紹介します。

[第1記事](https://zenn.dev/sakuyaxx21/articles/terraform-aws-infrastructure)ではAWS基盤とIaC、[第2記事](https://zenn.dev/sakuyaxx21/articles/github-actions-oidc-ssm)ではCI/CDを扱いました。

ソースコードは[Dev-PortfolioのGitHubリポジトリ](https://github.com/sakuyaxx21-sys/dev-portfolio)で公開しています。
以下の設定値は、現在の`infra/modules/monitoring`と`infra/modules/operations`のTerraform定義に基づきます。

## 今回構築した監視・通知

監視対象はASG、Target Group、ALB、RDSです。
8個のAlarmをcriticalとwarningに4個ずつ分け、それぞれ専用のSNS TopicとSlack Channel Configurationへ接続しています。

```text
ASG / Target Group / ALB / RDS のメトリクス
  └─ CloudWatch Alarm
       ├─ critical → critical用SNS Topic → Slack連携 → critical用チャンネル
       └─ warning  → warning用SNS Topic  → Slack連携 → warning用チャンネル
```

Slack連携にはAmazon Q Developer in chat applications（旧AWS Chatbot）を利用します。
Terraformのリソース名は`aws_chatbot_slack_channel_configuration`です。
SNS通知をチャットへ届ける役割と名称は[AWS公式ドキュメント](https://docs.aws.amazon.com/chatbot/latest/adminguide/what-is.html)で確認できます。

## 監視対象とメトリクス

CPUに加え、稼働台数、ターゲットの健全性、HTTPエラー、DBの空き容量と接続数を監視します。

| 対象 | メトリクス | 重要度 | 検知したい状態 |
| --- | --- | --- | --- |
| ASG | `GroupInServiceInstances`と`GroupDesiredCapacity`の差 | critical | InServiceの台数が希望台数を下回る |
| ASG配下のEC2 | `CPUUtilization` | warning | アプリケーションサーバ群のCPU高負荷 |
| Target Group | `UnHealthyHostCount` | critical | ALB配下で異常判定されたターゲットが存在する |
| Target Group | `HTTPCode_Target_5XX_Count` | warning | ターゲットが生成した5XXの継続発生 |
| ALB | `HTTPCode_ELB_5XX_Count` | critical | ALBが生成した5XXの継続発生 |
| RDS | `CPUUtilization` | warning | DBのCPU高負荷 |
| RDS | `FreeStorageSpace` | critical | DBの空きストレージが少ない |
| RDS | `DatabaseConnections` | warning | DB接続数の増加 |

### 台数とヘルスチェックは異なる状態を見る

ASGの台数監視では、CloudWatchのメトリクス数式で次の差を計算します。

```text
GroupInServiceInstances - GroupDesiredCapacity < 0
```

両方のメトリクスを60秒・`Average`で評価し、その差が負ならcriticalとするAlarmを定義しています。
希望台数が2台なら、InServiceが1台でも差分は負になります。

一方、`UnHealthyHostCount`はALBから見たターゲットの健全性を扱います。
今回のTarget Groupのヘルスチェック先は`/api/v1/health`です。
必要台数を満たしているかと、リクエストの転送先が正常に応答できるかを、別のAlarmとして追えるようにしています。

### 5XXは発生元を分ける

`HTTPCode_Target_5XX_Count`はターゲットが生成した5XX、`HTTPCode_ELB_5XX_Count`はALB自身が生成した5XXを数えます。
Target側ならアプリケーションログ、ALB側ならターゲットの状態やALBからターゲットへの接続状況などを確認する手掛かりになります。
ただし、ALBが生成した5XXだからといって、原因がAWSのALBサービス自体にあると断定できるわけではありません。
メトリクスの定義は[AWSのALBメトリクス一覧](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-cloudwatch-metrics.html)に沿って区別しています。

### namespaceとdimensionsで監視対象を特定する

namespaceでメトリクスの所属を、`dimensions`で対象を特定します。

| メトリクスのグループ | namespace | dimensions |
| --- | --- | --- |
| ASG台数の2メトリクス | `AWS/AutoScaling` | `AutoScalingGroupName` |
| EC2 CPU | `AWS/EC2` | `AutoScalingGroupName` |
| Target Groupの2メトリクス | `AWS/ApplicationELB` | `LoadBalancer`、`TargetGroup` |
| ALB 5XX | `AWS/ApplicationELB` | `LoadBalancer` |
| RDSの3メトリクス | `AWS/RDS` | `DBInstanceIdentifier` |

ALBとTarget GroupのdimensionsにはARNのsuffixを渡します。
EC2 CPUはASG名で集約し、`Average`で評価する設定です。
個々のEC2の最大CPU使用率を監視する条件とは区別しています。

## critical / warningの重要度設計

criticalは、稼働台数不足、ターゲット異常、ALBの5XX、DB空き容量不足など、優先して状況を確認したい状態に割り当てています。
warningは、CPU高負荷、ターゲットの5XX、DB接続数増加など、負荷やエラーの状況を調べるきっかけとして扱います。

この分類はDev-Portfolioの設計判断です。
warningのターゲット5XXも利用者に影響し得るため、影響の有無を分類しているわけではありません。

各Alarmは一つの重要度を持ちます。
同じメトリクスにwarning / criticalの2段階の閾値は設定していません。

### 名前にも重要度を含める

Alarm名は`{env}-{project}-ops-alarm-{resource}-{metric}-{severity}`です。
たとえば`dev-portfolio-ops-alarm-asg-capacity-crit`となり、末尾には`crit`または`warn`を使います。
名前と通知先を揃えますが、実際の振り分けは各Alarmの`alarm_actions`で明示しています。

## 閾値と評価期間を組み合わせる

Alarmは、集計期間と閾値、違反が必要な回数を組み合わせて判定します。

`period`は集計期間（秒）、`statistic`はその期間の集計方法です。
集計値を`threshold`と`comparison_operator`で比較し、`evaluation_periods`（N期間）のうち`datapoints_to_alarm`（M個）が違反するとALARMになります。
`treat_missing_data`は評価に必要なデータが欠ける場合の扱いです。

### 現在の判定条件

`M`の「省略」はコードに`datapoints_to_alarm`を記述していないことを示します。
その場合はN個すべての違反を必要とする扱いです。

| Alarmの対象 | 集計 | period | 閾値違反の条件 | N | M |
| --- | --- | --- | --- | --- | --- |
| ASG台数差分 | 両メトリクスが`Average` | 60秒 | 差分 < 0 | 1 | 省略（1） |
| EC2 CPU | `Average` | 300秒 | ≥ 80% | 2 | 省略（2） |
| Target不健全数 | `Average` | 60秒 | ≥ 1台 | 5 | 5 |
| Target 5XX | `Sum` | 60秒 | ≥ 1件 | 3 | 3 |
| ALB 5XX | `Sum` | 60秒 | ≥ 1件 | 3 | 3 |
| RDS CPU | `Average` | 300秒 | ≥ 80% | 2 | 省略（2） |
| RDS空き容量 | `Average` | 300秒 | ≤ 2,147,483,648 bytes（2 GiB） | 1 | 省略（1） |
| RDS接続数 | `Average` | 300秒 | ≥ 80接続 | 2 | 省略（2） |

比較演算子は、台数差分が`LessThanThreshold`、空き容量が`LessThanOrEqualToThreshold`、残りが`GreaterThanOrEqualToThreshold`です。
ASGの数式内の集計方法は、通常の`statistic`ではなく`metric`ブロック内の`stat`で指定します。

### NとMを具体例で読む

Target 5XXでは、`evaluation_periods = 3`、`datapoints_to_alarm = 3`です。
データが各期間に揃っている場合、直近3期間のうち3期間すべてで「1分間の合計が1件以上」ならALARMになります。

| 古い順に並べた1分ごとの件数 | 違反した期間数 | 3 / 3の条件 |
| --- | --- | --- |
| 1 → 1 → 1 | 3 | 満たす |
| 3 → 0 → 0 | 1 | 満たさない |
| 1 → 0 → 1 | 2 | 満たさない |

「3分間で合計3件以上」とは異なります。
1分だけ3件発生しても、残り2期間が実測値の0なら条件を満たしません。
この表の0は欠損ではなく、説明用の実測値です。

複数期間を使うことで、短い変動と継続する状態を分けて評価します。
`period`は集計幅なので、「設定時間が経過した瞬間に必ずSlackへ届く」という到達時間の保証ではありません。
評価パラメーターの意味は[AWSのPutMetricAlarm API仕様](https://docs.aws.amazon.com/AmazonCloudWatch/latest/APIReference/API_PutMetricAlarm.html)も参照できます。

### 数値の意図と調整する単位

5XXの`threshold = 1`には、初期運用で少数のエラーも拾うために意図的に低くしている、というコメントがあります。
実際の判定は3 / 3なので、**1件でも発生すれば直ちに通知する設定ではなく、少数の5XXが継続する状態を捉える設定**です。

ASG台数差分の0は、「希望台数に足りているか」という比較基準です。
Target不健全数の1は、期間平均で1台以上の不健全ターゲットがある状態を判定します。
瞬間的に1台でも不健全なら必ず発報する、という条件ではありません。

CPU 80%、DB接続数80、空き容量2 GiBは、初期監視値として設定しています。
閾値・評価期間は、通常時のメトリクスや通知頻度に応じてTerraformから調整できる構成にしています。

DB接続数の80は割合ではなく接続数、空き容量の2 GiBは絶対値です。

### 欠損データはnotBreachingにする

全8個のAlarmで、`treat_missing_data = "notBreaching"`を設定しています。
これは、欠損を補って判定する必要がある場合、そのデータを閾値違反として扱わない設定です。

ただし「データが届かなければ、直ちにOK」と単純化はできません。
CloudWatchはN個より広い範囲から実データを取得する場合があり、必要数が揃えば欠損の補完を使わず評価します。
欠損時の詳細は[AWSの欠損データの扱い](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/alarms-and-missing-data.html)に記載されています。

## Terraformで判定条件と通知経路をつなぐ

### Alarmの代表例

`infra/modules/monitoring/alarms.tf`のTarget 5XX Alarmから、説明とコメントを省略して抜粋します。

```hcl
resource "aws_cloudwatch_metric_alarm" "target_5xx" {
  alarm_name          = "${local.name_prefix}-ops-alarm-target-5xx-warn"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = 3
  datapoints_to_alarm = 3
  metric_name         = "HTTPCode_Target_5XX_Count"
  namespace           = "AWS/ApplicationELB"
  period              = 60
  statistic           = "Sum"
  threshold           = 1
  treat_missing_data  = "notBreaching"

  alarm_actions = [var.sns_warning_alerts_topic_arn]

  dimensions = {
    LoadBalancer = var.alb_arn_suffix
    TargetGroup  = var.target_group_arn_suffix
  }
}
```

criticalのAlarmでは、通知先に`var.sns_critical_alerts_topic_arn`を指定します。

### SNS Topicを重要度ごとに作る

`infra/modules/operations/sns.tf`では、`aws_sns_topic.critical_alerts`と`aws_sns_topic.warning_alerts`を作成します。
名前はそれぞれ`${local.name_prefix}-ops-sns-alerts-critical`、`${local.name_prefix}-ops-sns-alerts-warning`です。

`operations/outputs.tf`で両TopicのARNを公開し、環境側の`main.tf`から`monitoring`へ渡します。
module呼び出し内の通知先部分は次のとおりです。

```hcl
sns_critical_alerts_topic_arn = module.operations.sns_critical_alerts_topic_arn
sns_warning_alerts_topic_arn  = module.operations.sns_warning_alerts_topic_arn
```

### SNSとSlackチャンネルを対応させる

`infra/modules/operations/chatbot.tf`のcritical側から抜粋します。タグは省略しています。

```hcl
resource "aws_chatbot_slack_channel_configuration" "critical_alerts" {
  configuration_name = "${local.name_prefix}-chatbot-slack-alerts-critical"
  iam_role_arn       = aws_iam_role.chatbot.arn
  slack_team_id      = var.slack_team_id
  slack_channel_id   = var.slack_critical_channel_id
  sns_topic_arns     = [aws_sns_topic.critical_alerts.arn]

  logging_level = "ERROR"
}
```

warning側にも同じ種類のリソースがあり、`slack_warning_channel_id`とwarning用TopicのARNを指定します。
SlackワークスペースIDは共通の`slack_team_id`、チャンネルIDは重要度ごとに別入力です。

READMEの通知先は`#dev-portfolio-alerts-critical`と`#dev-portfolio-alerts-warning`です。
コードには名前ではなくIDを渡します。

この構成は、Channel Configurationの`sns_topic_arns`でSNSとの関連付けを定義します。

2個のChannel Configurationは同じ連携用IAM Roleを使用しています。
Slack側のワークスペース認可はサービス利用の前提となるため、Terraformのチャンネル設定と合わせて用意します。

## dev / prodで監視定義を共通化する

dev / prodでは同じmonitoring moduleを利用し、監視項目・閾値・評価期間・重要度を共通化しています。
今回の規模・用途では共通の監視ポリシーを使い、対象リソースを環境ごとに渡す構成にしました。
`local.name_prefix = "${var.env}-${var.project}"`によって、Alarm、SNS Topic、Slack連携設定の名前に環境名を含めます。

監視対象は各環境の`app`・`db`のoutputから、通知先ARNは`operations`のoutputから渡します。
閾値と評価期間は共通の`monitoring/alarms.tf`で管理し、Slack IDは環境側の入力変数として環境ごとに指定できます。

`monitoring/outputs.tf`ではAlarm名を重要度別のリストで公開しています。
監視設定をコードにすることで、閾値・違反回数・通知先を同じ差分で確認できます。
重要度を変える際には、Alarm名とSNSの参照が揃っているかもレビューできます。

## 通知対象の設計と代表的な発報確認

通知対象はALARMへの状態遷移に絞り、全8個のAlarmの`alarm_actions`に重要度別SNS TopicのARNを設定しています。
復旧時の`ok_actions`は、通知頻度とのバランスを考慮して設定していません。

SNSへのAlarmアクションは、状態が変わったときに実行されます。
ALARMが続く間、評価のたびに繰り返し通知する設定ではありません。
状態遷移の扱いは[AWSのCloudWatch Alarmの説明](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Alarms.html)を参照してください。
復旧は、CloudWatchの状態履歴とメトリクスから確認します。

### critical / warningそれぞれの通知先を確認する

代表的なAlarmで発報確認を行い、ALB 5XXのcritical通知とASG CPUのwarning通知が、それぞれ対応するSlackチャンネルへ到達することを確認しました。

| 発報確認に使用したAlarm | 重要度 | 到達を確認した通知先 |
| --- | --- | --- |
| `dev-portfolio-ops-alarm-alb-5xx-crit` | critical | critical用Slackチャンネル |
| `dev-portfolio-ops-alarm-asg-cpu-warn` | warning | warning用Slackチャンネル |

この確認により、Alarmに設定した重要度別SNS TopicとSlack通知先の対応を確かめました。

## まとめ

Dev-Portfolioでは、ASG / Target Group / ALB / RDSに8個のCloudWatch Alarmを定義し、critical / warning別のSNS TopicとSlack連携を実装しています。
複数のレイヤーを監視し、閾値と評価期間を組み合わせて、台数不足・健全性・エラー・負荷・容量を捉える構成にしました。

構築を通じて、メトリクスと検知したい状態を対応させることの重要性を理解しました。
Alarmの動作は、閾値だけでなく、集計方法、期間、N / M、欠損データの扱いまで含めて読み取る必要があります。

「何を検知するか」「どの条件で判断するか」「どこへ知らせるか」を一つの流れとして設計することが、基盤の状態を把握するための土台になると考えています。
