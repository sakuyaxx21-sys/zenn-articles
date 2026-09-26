---
title: "AWS WAFの誤検知でOAuth2認証がブロックされた原因と対処法"
emoji: "🛡️"
type: "tech"
topics: ["aws", "waf", "fastapi", "oauth2"]
published: false
---

## はじめに

Dev-Portfolioでは、AWS上で動かすFastAPIのSwagger UIからOAuth2認証を利用できるようにしています。
この認証でWAFの誤検知が発生し、トークン取得用のリクエストを通すためにTerraformの設定を修正しました。

対応の中心は、**SQLインジェクション対策のルールグループについて、特定のパスとHTTPメソッドを検査対象から除外すること**です。
本記事では、認証処理とWAFの役割を整理し、変更した条件と、その条件が保護範囲に与える影響を説明します。

基盤全体は[第1記事](https://zenn.dev/sakuyaxx21/articles/terraform-aws-infrastructure)、CI/CDは[第2記事](https://zenn.dev/sakuyaxx21/articles/github-actions-oidc-ssm)で紹介しています。
今回は[Dev-Portfolio](https://github.com/sakuyaxx21-sys/dev-portfolio)の実装とGit履歴をもとに、トラブルシューティングとWAFの調整に焦点を当てます。

## 問題が発生した構成

公開先はALBで、EC2上のDockerコンテナでFastAPIを実行し、ユーザー情報をRDS PostgreSQLに保存しています。
AWS WAFのWeb ACLはALBに関連付けています。

```text
Swagger UIを操作するブラウザ
  ↓ HTTPS
ALB（Web ACLによるWAF検査、TLS終端）
  ↓ 許可されたリクエストをHTTPで転送
EC2 / Docker / FastAPI
  ↓ ユーザー情報の取得
RDS PostgreSQL
```

WAFはALBとは別のプロキシサーバを配置したものではなく、ALBに関連付けたWeb ACLとしてリクエストを検査します。
WAFでブロックされたリクエストは、FastAPIの認証処理まで届きません。

関連付けは`infra/modules/app/app.tf`にあります。

```hcl
resource "aws_wafv2_web_acl_association" "alb" {
  resource_arn = aws_lb.app.arn
  web_acl_arn  = var.web_acl_arn
}
```

Security GroupはALBからEC2へのアプリケーションポートなど、レイヤー間の通信を制御します。
HTTPリクエストの内容を検査するWAF、リクエストを転送するALB、認証情報を照合するFastAPIでは、担当する処理が異なります。

## Git履歴に残っている問題と修正

認証機能の追加と、WAF誤検知への対応は別の変更として記録されています。

| 履歴 | 変更内容 |
| --- | --- |
| `4269e2b` / PR #14 | Swagger UI用のOAuth2認証を追加し、フォームを受け取る`/api/v1/auth/token`を実装 |
| `8e72302` / PR #35 | OAuth2トークン取得時のWAF誤検知に対応し、SQLインジェクション対策のルールグループに除外条件を追加 |

PR #35のマージコミットには「fix: Swagger OAuth2ログインのWAF誤検知を回避」と記録されています。
[修正コミット`8e72302`](https://github.com/sakuyaxx21-sys/dev-portfolio/commit/8e72302037d066302abdcebcb4404c50d30523ae)で変更されたファイルは、`infra/modules/security/waf.tf`だけです。
認証処理やGitHub ActionsのWorkflowは、この修正では変更していません。

一方、ローカルの履歴・ドキュメントには、当時のHTTPレスポンスやWAFログ、画面操作の記録は残っていません。
そのため、以下では実装から分かる切り分けの観点と修正内容を説明し、当時の調査順序や検知文字列は再現しません。

## OAuth2認証では何を送っているのか

### JSONログインとフォームによるトークン取得

FastAPI側には、同じ認証サービスを呼ぶ二つの入口があります。

| エンドポイント | リクエスト形式 | 認証情報のフィールド |
| --- | --- | --- |
| `POST /api/v1/auth/login` | `application/json` | `email`、`password` |
| `POST /api/v1/auth/token` | `application/x-www-form-urlencoded` | `username`、`password` |

Swagger UIのOAuth2 Password Flowで利用するのは、後者です。
`backend/app/api/v1/endpoints/auth.py`から、コメントを省略して抜粋します。

```python
@router.post("/token", response_model=TokenResponse)
def login_for_access_token(
    form_data: OAuth2PasswordRequestForm = Depends(),
    db: Session = Depends(get_db),
):
    token = login_service(
        db=db,
        email=form_data.username,
        password=form_data.password,
    )

    return {
        "access_token": token,
        "token_type": "bearer",
    }
```

`OAuth2PasswordRequestForm`はフォームから認証情報を受け取る依存関係です。
この実装では、フォームの`username`をメールアドレスとして`login_service`に渡します。
OAuth2 Password Flowが`username`と`password`をフォームフィールドとして送ることは、[FastAPIの公式説明](https://fastapi.tiangolo.com/tutorial/request-forms/)でも確認できます。

`login_service`はユーザーを取得し、保存されたパスワードハッシュとの照合に成功するとJWTを発行します。
JSONログインでも、このサービスを呼び出します。
つまり、アプリケーション内部の認証処理が共通でも、入口のパスと送信形式は異なります。

### トークン取得とBearer Tokenの検証を分ける

`backend/app/api/dependencies/auth.py`では、`OAuth2PasswordBearer`の`tokenUrl`を`/api/v1/auth/token`に設定しています。
これにより、Swagger UIが参照するトークン取得先をOpenAPIに定義しています。

認証付きAPIでは、`OAuth2PasswordBearer`でBearer Tokenを取得した後、`verify_token`で検証してユーザーを取得します。
今回調整したのは、JWTを取得するための`POST /api/v1/auth/token`です。
取得済みのJWTを使うすべてのAPIを、WAF検査から外したわけではありません。

## 原因の切り分け：どのレイヤーで止まるのか

### FastAPIの認証失敗とWAFのブロック

`backend/app/services/auth.py`では、ユーザーが見つからない場合やパスワードが一致しない場合に`InvalidCredentialsError`を送出します。
`backend/app/api/error_handlers.py`がこれをHTTP 401と次のJSONに変換します。

```json
{"detail": "Invalid email or password"}
```

これは、FastAPIまでリクエストが届き、認証情報を照合した結果です。
一方、WAFによるブロックは、その認証処理に入る前の判定です。

また、アプリケーション自身にも管理者権限不足時にHTTP 403を返す処理があります。
**HTTPステータスコードだけでWAFが原因だと判断せず、レスポンスの内容と、どのレイヤーが処理したかを対応させる必要があります**。

### 実装にあるログの確認先

この構成には、次のログ保存設定があります。

| 確認先 | 実装上の保存先・設定 | 切り分けで見る対象 |
| --- | --- | --- |
| WAFログ | `infra/modules/operations/logs.tf`のCloudWatch Logs連携 | WAFが適用したアクションとルール |
| ALBアクセスログ | `infra/modules/app/app.tf`のS3出力 | 公開入口で受け付けたリクエスト |
| アプリケーションログ | DockerログのCloudWatch Logs出力 | FastAPI側の処理・応答 |

WAFログでは、`action`、`terminatingRuleId`、`ruleGroupList`などからブロックと関係するルールを調べられます。
ルールグループ内の個別ルールは、`ruleGroupList`内の`terminatingRule`も確認対象になります。
各フィールドの意味は[AWSのWAFログ仕様](https://docs.aws.amazon.com/waf/latest/developerguide/logging-fields.html)に記載されています。

これは、この構成で原因を追う際の確認先です。
当時これらのログをどの順序で確認したか、どのログ値を得たかは、保存された履歴からは特定できません。

### ヘルスチェックが通ることと認証が通ること

ALBのTarget Groupは`/api/v1/health`をヘルスチェックに利用します。
アプリCDにも、SSMでEC2内の同パスを確認する処理と、GitHub Actionsから`APP_HEALTH_URL`へアクセスする処理があります。

これらは、トークン取得のフォームを送る処理とは異なります。
EC2内のlocalhostへのアクセスは公開入口のWAFを通らず、公開URLへのヘルスチェックもOAuth2ログインと同じパス・メソッド・ボディではありません。
ヘルスチェックと認証リクエストは、確認できる範囲を分けて捉える必要があります。

## WAFルールとOAuth2リクエストの関係

`infra/modules/security/waf.tf`では、次のAWSマネージドルールグループを設定しています。

| ルールグループ | priority | 今回の変更 |
| --- | --- | --- |
| `AWSManagedRulesCommonRuleSet` | 1 | 変更なし |
| `AWSManagedRulesSQLiRuleSet` | 2 | パスとHTTPメソッドによる除外条件を追加 |

SQLiはSQLインジェクションを指します。
`AWSManagedRulesSQLiRuleSet`は、リクエストに含まれるSQLインジェクションのパターンを検査するルールグループです。
検査対象にはボディだけでなく、クエリ引数やURIパスなども含まれます。
詳細は[AWSのルールグループ一覧](https://docs.aws.amazon.com/waf/latest/developerguide/aws-managed-rule-groups-use-case.html)を参照してください。

WAFの判定と、アプリケーションがその入力を正当な認証情報として処理できるかは別のものです。
フォームとして正しい形式のリクエストでも、WAFの検知条件に一致すれば、認証処理へ進む前にブロックされ得ます。

今回の履歴から分かるのは、OAuth2トークン取得の誤検知に対して、SQLiルールグループの適用範囲を変更したことです。
グループ内のどの個別ルールが反応したか、どのフィールドや文字列が一致したかは確認できません。
そのため、「パスワードの特定記号が原因だった」「フォーム形式自体がSQLインジェクションと判定された」といった説明はできません。

## Terraformで誤検知への対処を実装する

### 変更前と変更後

変更前のSQLiルールグループには、対象リクエストを絞る条件がありませんでした。
修正では`managed_rule_group_statement`の中に`scope_down_statement`を追加しています。

scope-down statementは、そのルールグループが検査するリクエストの範囲を絞る条件です。
条件に一致したリクエストだけが、グループ内の検査対象になります。
この動作は[AWSのscope-down statementの説明](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-scope-down-statements.html)に沿っています。

今回の条件を式にすると、次のとおりです。

```text
SQLiルールグループの検査対象
  = NOT（URIパスが /api/v1/auth/token と完全一致 AND メソッドが POST）
```

除外したいリクエストを`AND`で特定し、全体を`NOT`で反転させています。
`NOT`があるため、該当するトークン取得リクエストはscope-down statementに一致せず、SQLiルールグループの検査対象から外れます。

### 実際の設定

`infra/modules/security/waf.tf`のSQLiルールから、`statement`部分を抜粋します。

```hcl
statement {
  managed_rule_group_statement {
    name        = "AWSManagedRulesSQLiRuleSet"
    vendor_name = "AWS"

    scope_down_statement {
      not_statement {
        statement {
          and_statement {
            statement {
              byte_match_statement {
                search_string         = "/api/v1/auth/token"
                positional_constraint = "EXACTLY"

                field_to_match {
                  uri_path {}
                }

                text_transformation {
                  priority = 0
                  type     = "NONE"
                }
              }
            }

            statement {
              byte_match_statement {
                search_string         = "POST"
                positional_constraint = "EXACTLY"

                field_to_match {
                  method {}
                }

                text_transformation {
                  priority = 0
                  type     = "NONE"
                }
              }
            }
          }
        }
      }
    }
  }
}
```

`EXACTLY`で完全一致を指定し、`text_transformation`は`NONE`です。
`/api/v1/auth/`配下全体や、すべてのPOSTリクエストを除外する条件にはしていません。

条件の読み方を例にすると、次のようになります。
これは実リクエストの試験結果ではなく、Terraformの条件から整理した表です。

| リクエスト | SQLiルールグループの検査対象 |
| --- | --- |
| `POST /api/v1/auth/token` | 対象外 |
| `GET /api/v1/auth/token` | 対象 |
| `POST /api/v1/auth/login` | 対象 |
| `POST /api/v1/auth/token/` | 対象 |

「対象」は、WAFの評価がこのルールグループまで進んだ場合を指します。
また、WAFの検査対象であることと、そのパス・メソッドをFastAPIが受け付けることは別です。

### 保護範囲はどこまで変わったか

修正後もWeb ACLとALBの関連付けは維持し、Commonルールグループにも変更を加えていません。
両グループの`override_action`は`none`のままで、グループの判定をCountへ変更する対応でもありません。

この実装では、変更するルールグループとリクエストの条件を絞ることで、他のパス・メソッドに対するSQLi検査と、Commonルールグループによる検査を維持しています。
これが、WAF全体の検査を止めずに正常な認証通信へ対応したポイントです。

ただし、**対象リクエストではSQLiルールグループ全体が検査対象外になります**。
特定の個別ルールや、ボディの一部分だけを除外する設定ではありません。
Content-Typeや送信元を限定する条件もなく、このパスとメソッドに一致するリクエストへ適用されます。

FastAPI側のパスワード照合や、認証付きAPIのJWT検証は引き続き行われます。
WAFの検査対象を調整しても、認証を省略する処理は追加していません。
一方、認証処理があることを、除外したSQLi検査と同等の保護として扱うこともできません。

## 修正後の状態と確認できる範囲

PR #35で修正がマージされ、現在のTerraformにも同じ除外条件が残っています。
ただし、修正後にAWS上で再送したリクエストの結果や、Terraform applyの実行記録は、ローカルのリポジトリにはありません。
ここでは、HTTP 200の実測値や、特定環境での再試験成功を記載しません。

アプリケーション側では、`backend/tests/test_auth.py`に、フォームでトークンを取得し、Bearer Tokenで`/api/v1/users/me`を呼ぶテストがあります。
この正常系テストは、WAF修正より前のOAuth2対応時に追加されています。
現在は、不明なメールアドレスや誤ったパスワードでHTTP 401を期待するテストもあります。

これらはFastAPIのTestClientによるテストで、AWS上のWAFやALBは通りません。
テストコードの存在と、WAF経由で認証できたという実測結果は区別する必要があります。

GitHub ActionsのCIにはpytestとTerraformのfmt / validateがあり、WAF設定の適用を担う`Terraform CD`は手動実行です。
アプリCDでコンテナを更新する処理とは分かれています。
今回の修正対象がTerraformだけであることからも、アプリケーションの更新とWeb ACLの設定反映は別の操作として捉えられます。

## 今回のトラブルシューティングから得た理解

### 認証の入口まで含めて考える

ログインできない場合、メールアドレス・パスワードの照合だけが原因候補とは限りません。
今回の構成では、WAFの検査を通過し、ALBからFastAPIに転送されて初めて、アプリケーションの認証処理が始まります。
リクエストの経路と応答元を分けることが、調査対象を絞る軸になります。

### 正常な通信も、具体的なリクエストとして捉える

「ログイン」という機能名が同じでも、JSONログインとOAuth2のトークン取得では、パスやボディの形式が異なります。
WAFとの関係を考えるには、パス、HTTPメソッド、送信形式まで具体化する必要があります。
ヘルスチェックやアプリケーション単体のテストが、どの経路を確認しているかも同じ観点で整理できます。

### 例外設定は、残る検査と外れる検査を説明する

今回のTerraformは、SQLiルールグループと`POST /api/v1/auth/token`の組み合わせに例外を限定しています。
その一方で、対象リクエストに対してはグループ全体の検査が外れることも、設定の意味として押さえる必要があります。

誤検知への対応では、正常な通信を通す条件と、引き続き適用する検査をコードから説明できることが重要です。
Terraformに条件を残すことで、その境界を差分として追えるようになります。

## まとめ

Dev-Portfolioでは、Swagger UIのOAuth2トークン取得に対するWAF誤検知へ、`AWSManagedRulesSQLiRuleSet`の適用範囲を調整して対応しました。
`scope_down_statement`でパスとHTTPメソッドを組み合わせ、`POST /api/v1/auth/token`を対象外にしています。

この修正を理解するうえでは、WAFのリクエスト検査、ALBの転送、FastAPIの認証処理を分けて捉えることが重要でした。
正常な通信に必要な例外を具体的な条件で表現し、変更後の保護範囲まで説明することが、WAFを組み込んだアプリケーションの運用につながると考えています。
