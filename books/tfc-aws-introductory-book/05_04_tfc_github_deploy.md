---
title: "　Terraform CloudとGitHubリポジトリを接続してデプロイを実行"
---

:::message
本節では、前節で作成したTerraformコードを使用して、Terraform Cloud経由でリソースをデプロイします。

:::

## Projectの作成

Terraform Cloudで、Projectを作成します。

![](/images/chapter_5/04-01-project-01.png)

Project nameは以下とします。

| 項目  |  設定値  |
| ---- | ---- |
|  Project name  |  aws-tfc-introductory-book  |

![](/images/chapter_5/04-01-project-02.png)

## Variables Setの作成

AWSリソースをProject配下のWorkspaceが操作できるように、Terraform Cloud用のIAM Role ARNを持つVariables Setを作成します。

作成したVariables SetをProjectに割り当てることで、配下のWorkspaceでAWSリソースの操作ができるようになります。

IAM Role ARNは前の節「Terraform Cloud用IAMロール作成」で使用したARNと同じです。

AWS CLIで確認する場合は、以下のコマンドで確認できます。

```bash
aws iam get-role --role-name tfc-role --query Role.Arn --output text
```

Terraform Cloudのポータルサイト上で、`Settings` -> `Variables sets`の順に選択します。

以下のように必要な項目を設定して、Variables setsを作成します。

| 項目  |  設定値  |
| ---- | ---- |
|  Variables sets name  |  aws-tfc-introductory-book  |
|  Variables sets scope  |  aws-tfc-introductory-book  |

| Variable category  |  Key  |  Value  |  Sensitive  |
| ---- | ---- | ---- | ---- |
|  Environment variable  |  TFC_AWS_PROVIDER_AUTH  |  true  | No  |
|  Environment variable  |  TFC_AWS_RUN_ROLE_ARN  |  <`role_arn`>  |  No  |

![](/images/chapter_5/04-02-variables-01.png)

## Workspaceの作成(VCS Driven)

作成したProject配下にWorkspaceを作成します。

![](/images/chapter_5/04-03-workspace-01.png)

今回はGitHubのイベントをトリガーに各種Terraformコマンドを実行したいため、`Version control workflow`を選択します。

GitHubを選択して、サンプルコードをPushしたリポジトリを選択します。

![](/images/chapter_5/04-03-workspace-02.png)

![](/images/chapter_5/04-03-workspace-03.png)

Workspace名は以下を設定します。STGとPRODでWorkspaceを2つ作成します。

| 項目  |　設定値 |
| ---- | ---- |
| PROD環境|  prod-aws-tfc-introductory-book  |
| STG環境 |  stg-aws-tfc-introductory-book  |

![](/images/chapter_5/04-03-workspace-04.png)

Project配下に以下2つのWorkspaceが作成できたらOKです。

![](/images/chapter_5/04-03-workspace-05.png)

## Workspaceの設定

Runを実行する前に、Workspaceにて以下の作業を行う必要があります。

1. Terraform 実行ディレクトリの設定
2. STG WorkspaceのAuto Applyの有効化

### 1. Terraform 実行ディレクトリの設定

どのディレクトリでTerraformを実行するか指定する必要があります。

今回の場合は、tfファイルが`infra/chapter5/<prod or stg>`にあります。

このディレクトリを実行ディレクトリとして設定します。

`Workspace名`　-> `Settings` -> `General`の順に選択して、`Terraform Working Directory
`に設定します。

| Workspace名  |  実行ディレクトリ  |
| ---- | ---- |
|  prod-aws-tfc-introductory-book  |  infra/chapter5/prod  |
|  stg-aws-tfc-introductory-book  |  infra/chapter5/stg  |

![](/images/chapter_5/04-04-workspace-setting-1.png)

### 2. STG WorkspaceのAuto applyの有効化

STG Workspaceは手動承認無しで、mergeされたらDeployしたいためAuto Applyを有効にします。

`Workspace名` -> `Settings` -> `General`の順に選択して、`Apply Method`を`Auto apply`に変更します。

![](/images/chapter_5/04-04-workspace-setting-2.png)

## 動作確認

### Terraform Cloudのコンソールからデプロイ

リソースデプロイの準備ができました。実際にデプロイしてみましょう。

まずは、Terraform Cloudのコンソールからデプロイを実行してみます。

`Workspace`の`Runs` -> `Actions` -> `Start new run` で実行できます。

![](/images/chapter_5/04-05-manual-run-01.png)

任意でRunの実行理由を設定できます。

後で確認するときに、便利なため本番運用時はできるだけ設定しましょう。

今回は省略して、`Start run`を実行します。

![](/images/chapter_5/04-05-manual-run-02.png)

`Confirm & Apply`を選択して、Applyを実行します。

![](/images/chapter_5/04-05-manual-run-03.png)

Applyが成功したら、AWS上でリソースが作成されていることを確認できます。

![](/images/chapter_5/04-05-manual-run-04.png)

![](/images/chapter_5/04-05-manual-run-05.png)

### GitHubでPRを作成、マージしてデプロイ

mainブランチにPull Requestを出して、自動デプロイを試してましょう。

EC2インスタンスにEnvタグを付与するPull Requestです。

![](/images/chapter_5/04-06-auto-run-01.png)

Pull Requestを作成したタイミングで、PRODとSTGのWorkspaceでPlanが実行されます。

GitHub上に表示される`Check`の`Details`からTerraform CloudのPlan結果にアクセスできます。

![](/images/chapter_5/04-06-auto-run-02.png)

Pull Requestをマージすることで、`Auto apply`を設定のSTG Workspaceは自動でデプロイが実行されます。

PROD Workspaceは`Manual apply`設定のためが必要なため、この時点ではデプロイが実行されません。

![](/images/chapter_5/04-06-auto-run-03.png)

EC2のタグの状態を確認してみても、STGだけ追加されていることが分かります。

```bash
$ aws ec2 describe-tags --filters "Name=resource-type,Values=instance"
{
    "Tags": [
        {
            "Key": "Name",
            "ResourceId": "i-XXXXXXXXXX",
            "ResourceType": "instance",
            "Value": "prod-tfc-aws-book"
        },
        {
            "Key": "Env",
            "ResourceId": "i-YYYYYYYYYYY",
            "ResourceType": "instance",
            "Value": "stg"
        },
        {
            "Key": "Name",
            "ResourceId": "i-YYYYYYYYYYY",
            "ResourceType": "instance",
            "Value": "stg-tfc-aws-book"
        }
    ]
}
```

PROD Workspaceで手動承認を行うことでデプロイが実行されます。

![](/images/chapter_5/04-06-auto-run-04.png)

```bash
$ aws ec2 describe-tags --filters "Name=resource-type,Values=instance"
{
    "Tags": [
        {
            "Key": "Env",
            "ResourceId": "i-08fd0aa16711a4601",
            "ResourceType": "instance",
            "Value": "prod"
        },
        {
            "Key": "Name",
            "ResourceId": "i-08fd0aa16711a4601",
            "ResourceType": "instance",
            "Value": "prod-tfc-aws-book"
        },
        {
            "Key": "Env",
            "ResourceId": "i-0ed77b602855ef9fc",
            "ResourceType": "instance",
            "Value": "stg"
        },
        {
            "Key": "Name",
            "ResourceId": "i-0ed77b602855ef9fc",
            "ResourceType": "instance",
            "Value": "stg-tfc-aws-book"
        }
    ]
}
```

### 後片付け

### リソース削除

Terraform Cloud上でTerraformで作成したリソースの削除ができます。

`Workspace(<prod/stg>-aws-tfc-introductory-book)` -> `Settings` -> `Destruction and Deletion`を選択します。

`Queue destroy plan`を選択します。

![](/images/chapter_5/04-07-destroy-01.png)

確認画面がでるため、Workspace名を入力して`Queue destroy plan`を選択することでDestroy用のRunが行われます。

Workspaceから`Runs`を選択すると、Destroy用のRunが実行されていることを確認できます。

![](/images/chapter_5/04-07-destroy-02.png)

Runの実行が完了すると、実際にリソースが削除できていることを確認できます。

### Workspace削除

Workspaceの方も削除しておきます。

先程と同様に`Workspace(<prod/stg>-aws-tfc-introductory-book)` -> `Settings` -> `Destruction and Deletion`を選択します。

`Delete Workspace`の項目の`Delete from Terraform Cloud`を選択することで、Workspaceの削除が可能です。

![](/images/chapter_5/04-07-destroy-03.png)