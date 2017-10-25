# teset_codedeploy
参考 
https://qiita.com/hardreggaecafe/items/7ff3ada575a1100fa4a2
https://dev.classmethod.jp/cloud/circleci-codedeploy/

## gitに新しいリポジトリを切るよ
### git作成
* ローカルにクローン

## デプロイ用S3バケットの作成するよ
* バケット名： teset-codedeploy-app

## DEMOでCodeDeployを作成するよ
* Blue/Green と インプレース デプロイの作成でデモが出来るやっとこう
参考：http://aws.typepad.com/sajp/2015/12/what-is-blue-green-deployment.html
* 

## CircleCI用のIAMユーザの作成
* ユーザ：circleci-user  
* アクセスの種類：プログラムによるアクセス
* アクセス権限：既存のポリシーを直接アタッチ
    * ポリシーの作成
        * 独自のポリシーを作成
        * ポリシー名： circle_ci_s3_policy
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject"
            ],
            "Resource": [
                "arn:aws:s3:::teset-codedeploy-app/*"
            ]
        }
    ]
}
``` 
[^comment]:　versionは変えない
        * ポリシー名： circle_ci_codedeploy_policy
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "codedeploy:RegisterApplicationRevision",
        "codedeploy:GetApplicationRevision"
      ],
      "Resource": [
        "*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "codedeploy:CreateDeployment",
        "codedeploy:GetDeployment"
      ],
      "Resource": [
        "*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "codedeploy:GetDeploymentConfig"
      ],
      "Resource": [
        "*"
      ]
    }
  ]
}
```

* アクセスキー： AKIAJHLUIK5O3YQ5SYTA
* シークレットアクセスキー： gDvkDmQlbpQheiMJHAPBKWAbDxSao70TwUoIOxtS


## CircleCIへIAMユーザを登録
https://circleci.com/

### プロジェクト追加します
### INSIGHTからリポジトリの設定編集
* Project Settings > AWS keysより、先ほど取得したIAMユーザのアクセスキーを登録します。
    * 

## デプロイパラメータの設定
* circle.ymlを作成
```yml
test:
  override:
    - exit 0
deployment:
  staging:
    branch: master
    codedeploy:
      CodeDeployDemoApplication:
        application_root: /
        region: ap-northeast-1
        revision_location:
          revision_type: S3
          s3_location:
            bucket: teset-codedeploy-app
            key_pattern: CodeDeployDemoApplication-{BRANCH}-{SHORT_COMMIT}
        deployment_group: CodeDeployDemoFleet
        deployment_config: CodeDeployDefault.AllAtOnce
```

## masterにpushしてみる
* pushしたら成功した

## deploy groupを修正してデプロイしてみた
* 既存のEC2インスタンスに指定してやってみた
    * エラーになった
    * たぶん、CodeDeployのエージェントがないせい
    * インストールしてみる
```bash
$ rpm -qi codedeploy-agent
package codedeploy-agent is not installed
```

```
sudo yum update
cd /home/ec2-user
aws s3 cp s3://aws-codedeploy-ap-northeast-1/latest/install . --region ap-northeast-1
chmod +x ./install
sudo ./install auto
```

### エラーになりました
```
$ aws s3 cp s3://aws-codedeploy-ap-northeast-1/latest/install . --region ap-northeast-1
fatal error: Unable to locate credentials
```
IAM RoleをEc2に付与する必要があるそうです
* EC2一覧から対象のEC2を右クリックしてIAM ロールの割当/置換
* 先ほどDEMOで作ったIAMロールを割り当て
* 完了でさっきのコマンド再実行
* やっぱりエラー codedeployのデプロイのイベント表示で詳細見たら scripts/install_dependencies yum httpdでエラーだった
* エラー個所をコメアウトして再実行→成功！


## 注意点
codedeploy設置先には何もない状態からスタートすること、初回のファイルは上書きが出来ないのでエラーに必ずなる