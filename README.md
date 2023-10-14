# CircleCIに環境変数を設定する手順
1. CircleCIにログインする。
2. 左サイドバーの「Projects」をクリックする。
3. 対象のプロジェクトにある「・・・」をクリックする。
4. 「Project Settings」をクリックする。
5. 左サイドバーの「Environment Variables」→「Add Environment Variable」をクリックする。
6. 「Name」に`AWS_REGION`などを設定し、「Value」に`us-east-1`などを指定する。
7. 「Add Enviroment Variable」をクリックする。

## 「Name」に設定するものの例
リージョン　`AWS_REGION`

アクセスキー　`AWS_ACCESS_KEY_ID`

シークレットアクセスキー　`AWS_SECRET_ACCESS_KEY`

# CircleCIでスタックをデプロイする手順
1. 上記の「CircleCIに環境変数を設定する手順」に沿ってスタックをデプロイしたいリージョンを登録する。
2. スタックを作成するときに必要なポリシーをIAMユーザーにアタッチする。スタックを作成するのでCloudFormationを操作するためのポリシーは必ず必要(例：CloudFormationFullAccessなど)。ここで適切なポリシーがアタッチされたIAMユーザーのアクセスキーとシークレットアクセスキーをCircleCIの環境変数に登録する。
3. GitHubにリポジトリを作成する。
4. `git clone [リポジトリのURL]`を実行する。
5. `cd [リポジトリ名]`を実行する。
6. Windowsのエクスプローラーからクローンしたリポジトリのフォルダを開いて、`.circleci/config.yml`を作成する。
7. 同じくクローンしたリポジトリのフォルダにCloudFomationテンプレートのフォルダとファイルを作成する。
8. `config.yml`ファイルにCircleCIからスタックをデプロイするためのコードを記述する。
9. `git add .`を実行する。
10. `git commit -m "任意のメッセージ"`を実行する。
11. `git push origin [ブランチ名]`を実行する。
12. CircleCIにログインするとスタックが作成中であることが確認できる。

# CircleCIからスタックをデプロイする`config.yml`ファイルのコード解説
## ソースコード
```yaml
version: 2.1

orbs:
  aws-cli: circleci/aws-cli@4.0

executors:
  my-executor:
    docker:
      - image: circleci/python:3.10

jobs:
  deploy-vpc:
    executor: my-executor
    steps:
      - checkout
      - aws-cli/install
      - run:
          name: Deploy CloudFormation VPC Stack
          command: |
            aws cloudformation deploy \
              --template-file cloudformation/picture-upload-vpc.yml \
              --stack-name raise13-vpc \
              --parameter-overrides Prefix=raise13-vpc

  deploy-ec2:
    executor: my-executor
    steps:
      - checkout
      - aws-cli/install
      - run:
          name: Deploy CloudFormation EC2 Stack
          command: |
            aws cloudformation deploy \
              --template-file cloudformation/picture-upload-ec2.yml \
              --stack-name raise13-ec2 \
              --parameter-overrides Prefix=raise13-ec2 VPCPrefix=raise13-vpc

  deploy-rds:
    executor: my-executor
    steps:
      - checkout
      - aws-cli/install
      - run:
          name: Deploy CloudFormation RDS Stack
          command: |
            aws cloudformation deploy \
              --template-file cloudformation/picture-upload-rds.yml \
              --stack-name raise13-rds \
              --parameter-overrides Prefix=raise13-rds VPCPrefix=raise13-vpc EC2Prefix=raise13-ec2
          no_output_timeout: 20m

  deploy-alb:
    executor: my-executor
    steps:
      - checkout
      - aws-cli/install
      - run:
          name: Deploy CloudFormation ALB Stack
          command: |
            aws cloudformation deploy \
              --template-file cloudformation/picture-upload-alb.yml \
              --stack-name raise13-alb \
              --parameter-overrides Prefix=raise13-alb VPCPrefix=raise13-vpc EC2Prefix=raise13-ec2

  deploy-s3:
    executor: my-executor
    steps:
      - checkout
      - aws-cli/install
      - run:
          name: Deploy CloudFormation S3 Stack
          command: |
            aws cloudformation deploy \
              --template-file cloudformation/picture-upload-s3.yml \
              --stack-name raise13-s3 \
              --parameter-overrides Prefix=raise13-s3

workflows:
  version: 2
  deploy:
    jobs:
      - deploy-vpc
      - deploy-ec2:
          requires:
            - deploy-vpc
      - deploy-rds:
          requires:
            - deploy-ec2
      - deploy-alb:
          requires:
            - deploy-rds
      - deploy-s3:
          requires:
            - deploy-alb
```

## 解説
### `version: 2.1`
CircleCIの設定ファイルのバージョン
### `orbs:`
特定の機能をインポートするために必要。Pythonでいう`import`のようなもの。
### `aws-cli: circleci/aws-cli@4.0`
Pythonでいうライブラリのようなもの。

