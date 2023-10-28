# CircleCIに環境変数を設定
1. CircleCIにログインする。
2. 左サイドバーの「Projects」をクリックする。
3. 対象のプロジェクトにある「・・・」をクリックする。
4. 「Project Settings」をクリックする。
5. 左サイドバーの「Environment Variables」→「Add Environment Variable」をクリックする。
6. 「Name」に`AWS_REGION`などを設定し、「Value」に`us-east-1`などを指定する。
7. 「Add Enviroment Variable」をクリックする。

# CircleCIでスタックをデプロイ
1. 上記の「CircleCIに環境変数を設定」に沿ってスタックをデプロイしたいリージョンを登録する。
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
### `executors:`
このセクション内に書いた内容をプログラムコードの他の行で再利用可能にしてくれるもの。セクション内で書いた内容を他の行で利用するときは`executors`ではなく、`executor`と書くことに注意する。
### `- image: circleci/python:3.10`
CircleCIがジョブを実行するための環境であるDockerイメージを指定する。Dockerイメージはアプリケーションと実行環境をパッケージ化するためのテンプレート。
### `jobs:`
ジョブの具体的な内容を書くセクション。
### `steps:`
このセクション内で書かれた内容は上から順番に実行される。
### `checkout`
リポジトリのコードを読み取るためのもの。
### `aws-cli/install`
コマンドラインからAWSを操作するために必要なAWS CLIをインストール。
### `name: Deploy CloudFormation VPC Stack`
以下のようにCircleCI上から確認することができる処理名。自由につけて良い。

![スクリーンショット 2023-10-14 115427](https://github.com/Hidetaka-Konishi/Raise_AWS_13/assets/142459457/efce98d3-7606-407d-acbd-d6183b71584d)

### `aws cloudformation deploy \`
スタックをデプロイすることを定義している。
### `no_output_timeout: 20m`
CircleCIは`.circleci/config.yml`で出力がない状態が10分を経過するとタイムアウトしてしまうので、その期限を20分に設定している。RDSは作成されるまでの時間が長いため10分ではタイムアウトする可能性がある。
### `workflows:`
ジョブの連携を定義するセクション。
### `version: 2`
ワークフローのバージョン。
### `deploy:`
以下のようにCircleCI上で表示されるワークフローの名前。自由につけて良い。

![スクリーンショット 2023-10-14 154852](https://github.com/Hidetaka-Konishi/Raise_AWS_13/assets/142459457/a7fce6d8-2fd0-490e-b138-afcbf3b10260)

### `workflows:`の中の`jobs`
具体的にジョブがどのように実行されるのかを指定する場所。
### `requires:`
このセクション内に書かれた処理が終わったタイミングで`requires:`の一つ上の行の処理が実行される。

# Ansibleのインストール
1. Ubuntuで`sudo apt update`を実行する。
2. `sudo apt install ansible`を実行する。

# UbuntuからAnsibleのプレイブックをEC2に対して実行できるようにするための設定
1. Windowsのエクスプローラーの検索欄に`pem`と入力し、検索する。
2. pemファイルにカーソルを置いた状態で右クリックからコピーを選択する。
3. Ubuntuの`/home/[Ubuntuのユーザ名]/`ディレクトリにコピーしたpemファイルを貼り付ける。
4. Ubuntu上で`chmod 600 /home/[Ubuntuのユーザ名]/[pemのファイル名].pem`を実行する。

# Ansible内で良く使うコマンド
## プレイブックの構文チェック
`ansible-playbook -i [インベントリのファイル名].ini [プレイブックのファイル名].yml --syntax-check`
## Ubuntuからプレイブックの実行
`ansible-playbook -i [インベントリのファイル名].ini [プレイブックのファイル名].yml --key-file="/home/[ubuntuのユーザ名]/[pemのファイル名].pem"`
## プレイブックを実行して上手くいかなかった原因の詳細ログを知りたいとき
`ansible-playbook -i [インベントリのファイル名].ini [プレイブックのファイル名].yml -vvv`
## Vaultが設定されたプレイブックの構文チェック
`ansible-playbook -i [インベントリのファイル名].ini [プレイブックのファイル名].yml --syntax-check --ask-vault-pass`
## UbuntuからVaultが設定されたプレイブックの実行
`ansible-playbook -i [インベントリのファイル名].ini [プレイブックのファイル名].yml --key-file="/home/[ubuntuのユーザ名]/[pemのファイル名].pem" --ask-vault-pass`

# Ansible Vaultで個人情報を管理する。
1. `ansible-vault create [ファイル名].yml`を実行して新しく秘密ファイルを作成する。Vaultによって既に暗号化されているファイルを編集するときは`ansible-vault edit [ファイル名].yml`を実行する。
2. 秘密ファイルの中で`db_password: "supersecretpassword"`のように変数とパスワードを設定する。
3. 以下のようにプレイブック内で秘密ファイルの変数を使ってパスワードを参照することができる。`vars_files:`には秘密ファイルのパスを記述する。今回はこのプレイブックと同じディレクトリに`secret.yml`という秘密ファイルが存在するのでパスは`- secret.yml`となる。

```yaml
# playbook.yml
---
- hosts: your_server
  vars_files:
    - secret.yml
  tasks:
    - name: set db password
      environment:
        DB_PASSWORD: "{{ db_password }}"

```
