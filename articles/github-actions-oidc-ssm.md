---
title: "GitHub Actions × OIDC × SSMでAWSへのCI/CDを構築してみた"
emoji: "🚀"
type: "tech"
topics: ["githubactions", "aws", "oidc", "ssm"]
published: false
---

## はじめに

ポートフォリオ「Dev-Portfolio」として、Terraformで構築したAWS基盤上に、FastAPIを用いた社内向け申請管理システムを動かしています。
今回は、その基盤へGitHub ActionsからアプリケーションをデプロイするCI/CDを構築しました。

中心となるのは、OIDCによるAWS認証と、Systems Manager（SSM）Run CommandによるEC2上のコンテナ更新です。
AWSの長期アクセスキーをGitHubへ保存せず、EC2のSSHポートも開けない構成にしています。

第1記事「Terraformでdev/prodを分離したAWS Webアプリ基盤を構築してみた」では、基盤の構成と環境差分を扱いました。
本記事では、その上で「どのコードを、どの権限で、どのようにデプロイするか」を説明します。
<!-- 第1記事の公開URL確認後、上のタイトルにリンクを設定する。 -->

ソースコードは[Dev-PortfolioのGitHubリポジトリ](https://github.com/sakuyaxx21-sys/dev-portfolio)で公開しています。
イメージの保存先にはDocker Hubを使用しています。

## 今回構築したCI/CD

GitHub ActionsのWorkflowは、検証、アプリケーションの更新、インフラの変更に分けています。

| Workflow名 | ファイル | 実行契機 | 役割 |
| --- | --- | --- | --- |
| `CI` | `.github/workflows/ci.yml` | Pull Request、mainへのpush | pytest、Terraformのfmt / validate |
| `CD` | `.github/workflows/cd.yml` | mainで実行されたCIの完了、手動実行 | Docker imageのbuild / push、SSMデプロイ |
| `Terraform CD` | `.github/workflows/terraform-cd.yml` | 手動実行 | dev / prodのplan / apply |

`CD`の自動実行では、CIの結果が成功した場合にだけデプロイjobを動かします。
`workflow_dispatch`による手動実行も用意しています。

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
EC2へイメージ本体をSSM経由で転送する構成ではなく、SSMはEC2上で実行するコマンドを届けます。

## GitHub ActionsからAWSへOIDCで認証する

### 長期アクセスキーを保存しない認証

OIDC（OpenID Connect）を使うと、GitHub Actionsの実行元をAWS側で検証し、IAM Roleの一時的な認証情報を取得できます。
今回のAWS認証にはこの方式を採用し、IAM Userの長期アクセスキーをGitHub Secretsへ登録する必要をなくしています。

認証時には、GitHubが発行するOIDCトークンを使い、AWS STSの`AssumeRoleWithWebIdentity`でRoleを引き受けます。
取得した一時的な認証情報を、後続のAWS CLIが利用します。
この連携は[GitHubのAWS向けOIDC設定](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws)に沿ったものです。

AWS側の定義は`infra/modules/security/github_actions.tf`にあります。
OIDC ProviderのURLは`https://token.actions.githubusercontent.com`、audienceの既定値は`sts.amazonaws.com`です。
既存ProviderのARNが渡された場合はそれを参照し、未指定の場合だけProviderを作成します。
同じAWSアカウントでdev/prodを構築する際にも、既存Providerを共有できる形です。

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
`github_actions_repository`と`github_actions_branch`は入力変数であり、Terraformコードに特定の値を固定していません。
mainを許可する場合は、ブランチ変数に`main`を渡すことで、`ref:refs/heads/main`に対応します。

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
Secretsとして保存していますが、長期アクセスキーの代わりに認証能力を持つ秘密鍵を保存しているわけではありません。
Roleを引き受けるには、AWS側のTrust Policyを満たすOIDCトークンが必要です。

## CIでアプリケーションとTerraformを検証する

`CI`には3種類のjobを定義しています。

| job | 検証内容 |
| --- | --- |
| `Backend Test` | Python 3.11で依存関係をインストールし、`backend`で`pytest`を実行 |
| `Terraform Format` | `terraform fmt -check -recursive infra` |
| `Terraform Validate` | matrixでdev / prodそれぞれの構成を検証 |

バックエンドのテストは、`backend/tests/conftest.py`でSQLiteのテストDBとFastAPIの`TestClient`を用意しています。
AWS上のRDSへ接続するテストではなく、CI内でアプリケーションの動作を確認する構成です。

Terraformは`1.15.5`を使用し、validateの前に次の初期化を行います。

```yaml
- name: Initialize Terraform without backend
  run: terraform init -backend=false

- name: Validate Terraform configuration
  run: terraform validate
```

S3 Backendに接続せず、dev/prodそれぞれのroot moduleから構成を検証します。
共通moduleの変更も、両環境との入力・出力の関係を含めて確認できます。

CIではAWS認証やDockerイメージのbuild / pushは行いません。
これらは、検証後に配布・更新を行うCDへ分けています。

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

自動実行では、起点となったCIの`head_sha`を採用します。
これにより、CI終了後に別のコミットがmainへ追加されても、対象のコードを取り違えずにbuildできます。
手動実行では、その実行の`github.sha`を使用します。

手動実行は、上のjob条件ではCI成功を前提にしていません。
また、`workflow_dispatch`自体にmain限定の条件はなく、AWS認証時には別途Trust Policyのブランチ条件が評価されます。
Workflowの起動条件と、AWS操作を許可する条件を分けて捉えることが重要です。

## DockerイメージをDocker Hubへpushする

`backend/Dockerfile`は`python:3.11-slim`をベースに依存関係とアプリケーションを配置し、Uvicornを8000番ポートで起動します。
CDではDocker Hubへログインし、Buildxを設定してからbuild / pushします。

```yaml
- name: Log in to Docker Hub
  uses: docker/login-action@v4
  with:
    username: ${{ secrets.DOCKERHUB_USERNAME }}
    password: ${{ secrets.DOCKERHUB_TOKEN }}

- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v4

- name: Build and push Docker image
  uses: docker/build-push-action@v7
  with:
    context: backend
    file: backend/Dockerfile
    push: true
    pull: true
    no-cache: true
    tags: |
      ${{ env.DOCKER_IMAGE }}:${{ env.IMAGE_TAG }}
      ${{ env.DOCKER_IMAGE }}:latest
```

`DOCKER_IMAGE`にはGitHub Variablesの`DOCKER_IMAGE_NAME`を渡します。
`IMAGE_TAG`は前節のコミットSHAです。
`pull: true`でベースイメージの取得を試み、`no-cache: true`でbuildキャッシュを使わずにビルドします。

pushするタグは2つありますが、SSMによる更新ではSHAタグを指定します。
実行ログやイメージ名から、デプロイ対象のソースコードを追えるようにしています。

なお、ローカル開発用の`backend/docker-compose.yml`ではアプリケーションとPostgreSQLを起動しますが、EC2の更新処理ではComposeを使いません。
AWS上ではRDSへ接続し、コンテナは`docker run`で起動します。

## SSM Run CommandでEC2へデプロイする

### SSHを開けずにコマンドを実行する

要件定義では、EC2をPrivate App Subnetに配置し、運用接続にSystems Managerを使う方針としています。
Security GroupはALBからのアプリケーションポートだけを受け付け、22番ポートを許可していません。
CDにもSSMを使うことで、この通信方針を保ったままコンテナを更新できます。

EC2側には、Instance Profileを通じて`app_ec2` Roleを付与しています。
このRoleには`AmazonSSMManagedInstanceCore`を付け、SSM AgentがSystems Managerと通信できるようにしています。
SSM Agentはサービスからの要求を処理し、実行結果を返します。仕組みは[AWSのSSM Agentの説明](https://docs.aws.amazon.com/systems-manager/latest/userguide/ssm-agent.html)を参照してください。

AMIは通常版のAmazon Linux 2023を選択し、プリインストールされたSSM Agentを利用する構成です。
user dataでSSM Agentを追加インストールする処理はありません。
プリインストール対象は[AWSのAMI一覧](https://docs.aws.amazon.com/systems-manager/latest/userguide/ami-preinstalled-agent.html)で確認できます。
Private App SubnetからのAWS APIアクセスやイメージ取得には、NAT Gateway経由の外向き通信を利用します。

GitHub Actions側のRoleはコマンドの**送信側**、EC2側のRoleはSSMの管理対象として動く**実行側**です。
両者を分けて定義することで、WorkflowとEC2の責務を対応させています。

### Run Commandを送信する

Workflowからは`.github/scripts/deploy-ec2-via-ssm.sh`を呼び出します。
スクリプトは`jq`でコマンド列をJSONにし、一時ファイルへ保存してから送信します。
送信部分の抜粋です。

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

対象は`SSM_TARGET_KEY`と`SSM_TARGET_VALUE`で指定します。
WorkflowにインスタンスIDを固定せず、GitHub Variablesから対象条件を渡す構成です。

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
SSMの完了確認に加え、外側からもHTTP応答を確認することで、更新処理と公開経路をそれぞれ確認しています。

## Terraformのplan / applyは別のWorkflowで実行する

`Terraform CD`は`workflow_dispatch`専用です。
`environment`に`dev` / `prod`、`action`に`plan` / `apply`を選び、`infra/envs/${{ inputs.environment }}`で実行します。

AWS認証には、アプリCD用とは別の`AWS_TERRAFORM_ROLE_ARN`を使います。
初期化、fmt、validateの後に、次のstepを実行します。

```yaml
- name: Create Terraform plan
  run: terraform plan -out=tfplan

- name: Show Terraform plan
  run: terraform show -no-color tfplan

- name: Apply Terraform plan
  if: inputs.action == 'apply'
  run: terraform apply -auto-approve tfplan
```

`plan`を選ぶと実行計画の表示まで、`apply`を選ぶと同じ実行内で作成した`tfplan`を適用します。
別のplan実行で保存した成果物を、後から承認して適用する方式ではありません。
現在のWorkflowには`destroy`の選択肢やstepはありません。

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

ここでは、対象リポジトリの`dev` / `prod` Environmentを条件にしています。
WorkflowにもこのTrust Policyにも、Terraform実行をmainへ限定する条件はありません。
`TF_GITHUB_ACTIONS_BRANCH`はアプリCD用Roleの信頼条件へ渡す変数であり、Terraform Workflow自身の実行ブランチを制限するものではありません。

Terraformが担うのはAWSリソースとEC2の初回起動設定です。
アプリCDは、すでに起動しているEC2のコンテナ更新を担います。
日々のアプリケーション更新と基盤変更を、別のWorkflow・Roleで扱う構成です。

## CI/CDで意識した設定と権限の分離

設定値と認証情報は、用途に応じて管理場所を分けています。

| 管理場所 | 主な値 | 用途 |
| --- | --- | --- |
| GitHub Secrets | `DOCKERHUB_USERNAME`、`DOCKERHUB_TOKEN` | イメージのpush認証 |
| GitHub Secrets | `AWS_ROLE_ARN`、`AWS_TERRAFORM_ROLE_ARN` | OIDCで引き受けるRoleの指定 |
| GitHub Variables | `AWS_REGION`、`DOCKER_IMAGE_NAME` | AWSの操作先とイメージ名 |
| GitHub Variables | `SSM_TARGET_KEY`、`SSM_TARGET_VALUE` | Run Commandの対象指定 |
| GitHub Variables | `APP_CONTAINER_NAME`、`APP_ENV_FILE`、`APP_PORT`、`APP_HEALTH_URL` | コンテナ起動・確認設定 |
| GitHub Variables | `TF_PROJECT`、`TF_DOCKER_IMAGE_NAME`、`TF_DOCKER_IMAGE_TAG`など | Terraformの入力 |
| AWS Secrets Manager | DB認証情報、アプリケーションSecret | EC2上の実行時設定 |

Terraformでは、リポジトリ・ブランチ、ドメイン、Slack通知先も`TF_*`のVariablesから渡しています。
既存OIDC Providerを使う場合は、任意の`TF_GITHUB_ACTIONS_OIDC_PROVIDER_ARN`を追加します。
両CD Workflowは、必要な設定値が空の場合に処理を止めるチェックを持っています。

OIDCによって不要になるのは、AWS認証用の長期アクセスキーの保存です。
Docker Hubの認証情報や、アプリケーションの機密情報は引き続き必要です。

また、認証方式、Roleの信頼条件、Roleに付与する操作権限は、それぞれ確認する必要があります。
今回の実装では、アプリCD・Terraform・EC2でRoleを分け、用途に対応させています。

## CI/CDを構築して得た理解

### SSMにはIAM・通信経路・Agentの準備が必要

構築中、EC2へのSession Manager接続で`TargetNotConnected`が発生しました。
調査では、EC2の稼働状態、Instance Profile、`AmazonSSMManagedInstanceCore`、NAT Gatewayへの経路を確認し、そのうえでSSMへの登録状態と使用中のAMIを調べました。

原因は、AMIの検索条件が広く、SSM Agentを含まないMinimal AMIを選択していたことでした。
現在は`infra/modules/app/app.tf`で、次の条件に絞っています。

```hcl
filter {
  name   = "name"
  values = ["al2023-ami-2023*-x86_64"]
}
```

これはSession Managerで発見した問題ですが、Run CommandでもEC2がSSMの管理対象として利用できることが前提になります。
IAM Roleを付けることに加え、Agentと通信経路まで確認して初めて、SSM経由の操作につながると理解しました。

### 認証・成果物・実行結果をつなげて考える

認証については、OIDCの設定だけでなく、Workflowの実行コンテキストとTrust Policyの対応が重要でした。
アプリCDではブランチ、TerraformではEnvironmentを照合するため、同じGitHub Actionsからの実行でもRoleごとに条件が異なります。

デプロイについては、CIのコミットSHAをcheckoutとイメージタグへ引き継ぎ、Run Commandの結果を待ってから公開URLを確認しています。
「どのコードを更新したか」と「どこまで処理が完了したか」を追えることが、パイプラインを理解し、問題を切り分けるうえで重要だと感じました。

## まとめ

Dev-Portfolioでは、GitHub Actionsで検証したコードをDockerイメージとして配布し、OIDCとSSMを使ってAWS上のEC2へデプロイする構成を実装しました。
AWS認証には一時的な認証情報を使い、コンテナ更新にはSSHを開けずに実行できるRun Commandを利用しています。

アプリケーション更新とTerraformによる基盤変更を分け、それぞれのWorkflow、Role、実行条件を対応させました。
個々の自動化処理に加えて、実行元の認証から成果物の特定、更新結果の確認までをつなぐことが、CI/CDを設計するうえで大切だと考えています。
