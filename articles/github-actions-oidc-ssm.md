---
title: "GitHub Actions × OIDC × SSMでAWSへのCI/CDを構築してみた"
emoji: "🚀"
type: "tech"
topics: ["githubactions", "aws", "oidc", "ssm"]
published: false
---

## はじめに

ポートフォリオ「Dev-Portfolio」として、Terraformで構築したAWS基盤上に、FastAPIを用いた社内向け申請管理システムを動かしています。
GitHub Actionsからのデプロイには、OIDCによるAWS認証とSystems Manager（SSM）Run Commandによるコンテナ更新を組み合わせました。
AWSの長期アクセスキーをGitHubへ保存せず、EC2のSSHポートも開けない構成にしています。

第1記事「[Terraformでdev/prodを分離したAWS Webアプリ基盤を構築してみた](https://zenn.dev/sakuyaxx21/articles/terraform-aws-infrastructure)」では、基盤の構成と環境差分を扱いました。
本記事では、その上で「どのコードを、どの権限で、どのようにデプロイするか」を説明します。

ソースコードは[Dev-PortfolioのGitHubリポジトリ](https://github.com/sakuyaxx21-sys/dev-portfolio)で公開しています。
イメージの保存先にはDocker Hubを使用しています。

## 今回構築したCI/CD

Workflowは検証、アプリケーション更新、インフラ変更に分けています。

| Workflow名 | ファイル | 実行契機 | 役割 |
| --- | --- | --- | --- |
| `CI` | `.github/workflows/ci.yml` | Pull Request、mainへのpush | pytest、Terraformのfmt / validate |
| `CD` | `.github/workflows/cd.yml` | mainで実行されたCIの完了、手動実行 | Docker imageのbuild / push、SSMデプロイ |
| `Terraform CD` | `.github/workflows/terraform-cd.yml` | 手動実行 | dev / prodのplan / apply |

### アプリケーションを届ける流れ

通常の自動デプロイは、次の順序で進みます。

1. mainへのpushを受けて`CI`が実行されます。
2. CI成功後、`CD`がその実行のコミットをcheckoutします。
3. Dockerイメージをbuildし、コミットSHAと`latest`のタグでDocker Hubへpushします。
4. GitHub ActionsがOIDCでAWSのIAM Roleを引き受けます。
5. 取得した一時的な認証情報でSSM Run Commandを送信します。
6. EC2がSHAタグのイメージをpullし、マイグレーションとコンテナ更新を実行します。
7. コマンドの実行結果とヘルスチェックで完了を確認します。

ここには、**イメージの配布経路**と**AWSへの認証・操作経路**があります。
Docker HubへのpushにはDocker Hubの認証情報を使い、SSMの呼び出しにはOIDCで取得したAWS認証情報を使います。
SSMはEC2上で実行するコマンドを届け、EC2がDocker Hubからイメージを取得します。

## GitHub ActionsからAWSへOIDCで認証する

### 長期アクセスキーを保存しない認証

OIDC（OpenID Connect）を使うと、GitHub Actionsの実行元をAWS側で検証し、IAM Roleの一時的な認証情報を取得できます。
今回のAWS認証にはこの方式を採用し、IAM Userの長期アクセスキーをGitHub Secretsへ登録する必要をなくしています。

認証時には、GitHubが発行するOIDCトークンを使い、AWS STSの`AssumeRoleWithWebIdentity`でRoleを引き受けます。
取得した一時的な認証情報を、後続のAWS CLIが利用します。
この連携は[GitHubのAWS向けOIDC設定](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws)に沿ったものです。

AWS側の定義は`infra/modules/security/github_actions.tf`にあります。
OIDC ProviderのURLは`https://token.actions.githubusercontent.com`、audienceの既定値は`sts.amazonaws.com`です。
既存ProviderのARNが渡された場合はそれを共有し、未指定の場合だけProviderを作成します。

### Trust Policyで実行元を制限する

アプリCD用RoleのTrust Policyから、主要部分を抜粋します。

```hcl
Principal = {
  Federated = local.github_actions_oidc_provider_arn
}
Action = "sts:AssumeRoleWithWebIdentity"
Condition = {
  StringEquals = {
    "token.actions.githubusercontent.com:aud" = var.github_actions_oidc_audience
  }
  StringLike = {
    "token.actions.githubusercontent.com:sub" = "repo:${var.github_actions_repository}:ref:refs/heads/${var.github_actions_branch}"
  }
}
```

`aud`でトークンの対象を、`sub`でリポジトリとブランチを照合しています。
リポジトリとブランチは入力変数で指定します。ブランチに`main`を渡すと、`ref:refs/heads/main`を照合します。

Trust Policyは「誰がRoleを引き受けられるか」を定義します。
引き受けた後に「何を操作できるか」は、Roleに付与するIAM Policyで決まります。

### WorkflowからRoleを引き受ける

`cd.yml`のWorkflow全体に対する権限設定です。

```yaml
permissions:
  contents: read
  id-token: write
```

続いて、deploy jobから認証stepを抜粋します。

```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v6
  with:
    role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
    aws-region: ${{ env.AWS_REGION }}
```

`id-token: write`はOIDCトークンの取得を許可する設定です。
AWSリソースへの書き込み権限そのものを付与する設定ではありません。

`AWS_ROLE_ARN`は引き受け先の識別子です。
Secretsに保存したARNだけで認証できるわけではなく、Trust Policyを満たすOIDCトークンが必要です。

## CIでアプリケーションとTerraformを検証する

`CI`には3種類のjobを定義しています。

| job | 検証内容 |
| --- | --- |
| `Backend Test` | Python 3.11で依存関係をインストールし、`backend`で`pytest`を実行 |
| `Terraform Format` | `terraform fmt -check -recursive infra` |
| `Terraform Validate` | matrixでdev / prodそれぞれの構成を検証 |

Terraformは`1.15.5`を使用し、`terraform init -backend=false`の後に`terraform validate`を実行します。
S3 Backendに接続せず、dev/prodそれぞれのroot moduleから構成を検証します。

CIは検証を担当し、AWS認証、Dockerイメージのbuild / push、EC2へのデプロイはCDへ分けています。

## CIの結果とデプロイ対象のコミットをつなぐ

`cd.yml`のtriggerとjob条件は次のとおりです。job内の詳細は省略しています。

```yaml
on:
  workflow_run:
    workflows:
      - CI
    types:
      - completed
    branches:
      - main
  workflow_dispatch:

jobs:
  deploy:
    name: Deploy
    runs-on: ubuntu-latest
    if: github.event_name == 'workflow_dispatch' || github.event.workflow_run.conclusion == 'success'
```

`completed`は成功・失敗を問わず完了を表すため、job側で`conclusion == 'success'`を確認します。
`branches: main`は、起点となるCIの実行ブランチを絞る条件です。
イベントの意味は[GitHubのworkflow_runの説明](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows#workflow_run)で確認できます。

checkoutの`ref`とイメージタグには、どちらも次の式を使います。

```yaml
${{ github.event.workflow_run.head_sha || github.sha }}
```

自動実行ではCIの`head_sha`を採用し、その後mainに別のコミットが追加されても、検証済みのコードをbuildします。
手動実行では、その実行の`github.sha`を使用します。

`workflow_dispatch`による手動実行にも対応し、AWS操作時にはOIDCでRoleを引き受け、Trust PolicyとIAM Policyに従って操作します。

## DockerイメージをDocker Hubへpushする

CDではDocker Hubへログインし、`backend/Dockerfile`からイメージをbuild / pushします。
認証には`DOCKERHUB_USERNAME`と`DOCKERHUB_TOKEN`を使用します。
タグに関係する部分を抜粋します。

```yaml
- name: Build and push Docker image
  uses: docker/build-push-action@v7
  with:
    context: backend
    file: backend/Dockerfile
    push: true
    # その他のbuild設定は省略
    tags: |
      ${{ env.DOCKER_IMAGE }}:${{ env.IMAGE_TAG }}
      ${{ env.DOCKER_IMAGE }}:latest
```

`DOCKER_IMAGE`にはGitHub Variablesの`DOCKER_IMAGE_NAME`、`IMAGE_TAG`には前節のコミットSHAを渡します。
SHAと`latest`の2タグをpushし、SSMによる更新ではSHAタグを指定します。
これにより、CIで検証したコミット、buildするコード、EC2がpullするイメージを対応させています。

## SSM Run CommandでEC2へデプロイする

### SSHを開けずにコマンドを実行する

要件定義では、EC2をPrivate App Subnetに配置し、運用接続にSystems Managerを使う方針としています。
Security GroupはALBからのアプリケーションポートだけを受け付け、22番ポートを許可していません。
CDにもSSMを使うことで、この通信方針を保ったままコンテナを更新できます。

Run Commandを利用するには、EC2がSystems Managerの管理対象になっている必要があります。
EC2にはInstance Profileを通じて`app_ec2` Roleを付与し、`AmazonSSMManagedInstanceCore`でSSMに必要な通信を許可しています。
通常版のAmazon Linux 2023にプリインストールされたSSM Agentが、要求を処理して実行結果を返します。
Agentの役割は[AWSのSSM Agentの説明](https://docs.aws.amazon.com/systems-manager/latest/userguide/ssm-agent.html)を参照してください。
Systems Managerへの到達やイメージ取得には、Private App SubnetからNAT Gateway経由の外向き通信を利用します。

GitHub Actions側のRoleはコマンドの**送信側**、EC2側のRoleはSSMの管理対象として動く**実行側**です。
両者を分けて定義し、WorkflowとEC2の責務を対応させています。

### Run Commandを送信する

Workflowからは`.github/scripts/deploy-ec2-via-ssm.sh`を呼び出します。
`jq`でコマンド列をJSONファイルにし、次のコマンドで送信します。

```bash
aws ssm send-command \
  --region "$AWS_REGION" \
  --document-name "AWS-RunShellScript" \
  --targets "Key=${SSM_TARGET_KEY},Values=${SSM_TARGET_VALUE}" \
  --parameters "file://${params_file}" \
  --comment "Deploy ${full_image}" \
  --timeout-seconds "${SSM_COMMAND_TIMEOUT_SECONDS:-900}" \
  --query "Command.CommandId" \
  --output text
```

対象条件はGitHub Variablesの`SSM_TARGET_KEY`と`SSM_TARGET_VALUE`から渡します。

アプリCD用IAM Policyでは、`ssm:SendCommand`の対象を次のように定義しています。

```hcl
Action = [
  "ssm:SendCommand"
]
Resource = [
  "arn:aws:ssm:${data.aws_region.current.name}::document/AWS-RunShellScript",
  "arn:aws:ec2:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:instance/*"
]
```

実行するSSM Documentと、同一アカウント・リージョンのEC2を指定しています。
このほか、コマンド結果やインスタンス情報の取得権限を付与しています。
`--targets`は実行先を選ぶ設定であり、IAM Policyの許可範囲とは別です。

### EC2上でイメージとコンテナを更新する

Run Commandで実行する処理は、次の順序です。

1. `${DOCKER_IMAGE}:${IMAGE_TAG}`をpullします。
2. 新しいイメージを一時コンテナとして起動し、`alembic upgrade head`を実行します。
3. 既存アプリケーションコンテナを停止・削除します。
4. 同じ新しいイメージで常駐コンテナを起動します。
5. EC2内の`/api/v1/health`を確認します。

環境変数は`APP_ENV_FILE`で指定するEC2上のファイルを読み込みます。
このファイルは起動時のuser dataで、EC2 Roleを使ってSecrets Managerから取得した情報をもとに生成しています。
CDは既存のファイルを利用し、DBパスワードをGitHub Actionsから送信しません。

マイグレーションを既存コンテナの停止前に置き、pullやマイグレーションが失敗した場合は、その時点で処理を止める順序です。
正常に進むと、`--restart unless-stopped`を指定した新しいコンテナへ入れ替えます。

### コマンドの受付と完了を分けて確認する

`send-command`が返すCommand IDは、EC2上の更新完了を意味しません。
スクリプトでは、そのIDを使って`list-command-invocations --details`を繰り返し呼び出します。

まだ結果がない場合や、`Pending` / `InProgress` / `Delayed`が含まれる場合は、5秒待って再確認します。
最大120回確認し、取得した実行結果がすべて`Success`になった場合に成功とします。
それ以外の終了状態では、Instance ID、Status、CommandPluginsなどを出力して失敗させます。

ヘルスチェックは、確認場所も分けています。

| 確認場所 | 処理 | 確認する範囲 |
| --- | --- | --- |
| EC2内 | localhostの`/api/v1/health`を最大30回確認し、失敗時は5秒待つ | 起動したコンテナがHTTP応答を返すこと |
| GitHub Actions | `curl -fsS "$APP_HEALTH_URL"` | 設定した公開URLがHTTP応答を返すこと |

EC2内で応答が得られなければ、コンテナログの末尾100行を出力して失敗させます。
SSMの実行結果と内外のHTTP応答を組み合わせ、更新処理と公開経路を確認します。

## Terraformのplan / applyは別のWorkflowで実行する

`Terraform CD`は`workflow_dispatch`専用です。
`environment`に`dev` / `prod`、`action`に`plan` / `apply`を選び、`infra/envs/${{ inputs.environment }}`で実行します。

AWS認証はOIDCを使い、アプリCD用とは別の`AWS_TERRAFORM_ROLE_ARN`を指定します。
初期化、fmt、validateの後、`terraform plan -out=tfplan`で実行計画を保存して表示します。
`apply`を選んだ場合は、同じ実行内の`tfplan`を`terraform apply -auto-approve tfplan`で適用します。

### Terraform用RoleはEnvironmentを信頼条件にする

Terraformのjobは`environment: ${{ inputs.environment }}`を指定しています。
対応するTrust Policyの`sub`は、アプリCDのブランチ形式とは異なります。

```hcl
StringLike = {
  "token.actions.githubusercontent.com:sub" = [
    "repo:${var.github_actions_repository}:environment:dev",
    "repo:${var.github_actions_repository}:environment:prod"
  ]
}
```

対象リポジトリの`dev` / `prod` Environmentを条件として、Terraform用Roleを引き受けます。

Terraformが担うのはAWSリソースとEC2の初回起動設定です。
アプリCDは、すでに起動しているEC2のコンテナ更新を担います。

## CI/CDで意識した設定と権限の分離

設定値と認証情報は、次のように管理場所を分けています。

| 管理場所 | 管理する情報 |
| --- | --- |
| GitHub Secrets | Docker Hubの認証情報、OIDCで引き受けるRole ARN |
| GitHub Variables | リージョン、イメージ名、SSM対象、コンテナ・health check設定、Terraformの入力などの非機密情報 |
| AWS Secrets Manager | DB認証情報、アプリケーションSecret |

OIDCによって不要になるのはAWS認証用の長期アクセスキーの保存です。
Docker Hubの認証情報や、アプリケーションの機密情報は引き続き必要です。
両CD Workflowでは、必要な設定値が空の場合に処理を止めるチェックも行っています。

## まとめ

Dev-Portfolioでは、GitHub Actionsでの検証からDockerイメージの配布、OIDCとSSMによるEC2へのデプロイまでを実装しました。
アプリCD・Terraform・EC2でRoleを分け、アプリケーション更新と基盤変更を別のWorkflowで扱っています。

構築を通じて、認証・成果物・実行結果を一連のパイプラインとして考えることが重要だと理解しました。
OIDC認証では実行コンテキストとTrust Policyを対応させ、自動デプロイではCIのコミットSHAをcheckout、イメージタグ、EC2のpullへ引き継ぎます。
さらに、SSMの実行結果を待ち、localhostと公開URLのヘルスチェックで確認します。

「どの権限で、どのコードを更新し、どこまで処理が完了したか」を追えることが、CI/CDの設計と運用につながると考えています。
