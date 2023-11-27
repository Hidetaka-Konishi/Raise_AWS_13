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
2. スタックを作成するときに必要なポリシーをIAMユーザーにアタッチする。スタックを作成するのでCloudFormationを操作するためのポリシーは必ず必要(例：CloudFormationFullAccessなど)。ここで適切なポリシーがアタッチされたIAMユーザーのアクセスキーとシークレットアクセスキーをCircleCIの環境変数に保存する。CircleCIの環境変数に保存しているIAMユーザーにマネジメントコンソール上から追加でポリシーをアタッチすることで即そのポリシーがCircleCIのプロジェクトで反映される。
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
```
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
                        --parameter-overrides Prefix=raise13 \
                        --capabilities CAPABILITY_NAMED_IAM

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
                        --parameter-overrides Prefix=raise13 \
                        --capabilities CAPABILITY_NAMED_IAM

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
                        --parameter-overrides Prefix=raise13 \
                        --capabilities CAPABILITY_NAMED_IAM
                  no_output_timeout: 30m

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
                        --parameter-overrides Prefix=raise13 \
                        --capabilities CAPABILITY_NAMED_IAM

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
                        --parameter-overrides Prefix=raise13 \
                        --capabilities CAPABILITY_NAMED_IAM

    run-ansible:
        executor: my-executor
        steps:
            - checkout
            - aws-cli/install
            - run:
                  name: Retrieve Region of VPC
                  command: |
                      REGION=$(aws cloudformation describe-stacks \
                        --stack-name raise13-vpc \
                        --query 'Stacks[0].Outputs[?OutputKey==`RegionName`].OutputValue' \
                        --output text)
                      echo "export REGION=$REGION" >> $BASH_ENV
            - run:
                  name: Retrieve EC2 Key Pair
                  command: |
                      KEY_NAME=$(aws cloudformation describe-stacks \
                        --stack-name raise13-ec2 \
                        --query 'Stacks[0].Outputs[?OutputKey==`KeyName`].OutputValue' \
                        --output text)
                      echo "export KEY_NAME=$KEY_NAME" >> $BASH_ENV
            - run:
                  name: Retrieve Public IP of EC2 Instance
                  command: |
                      PUBLIC_IP=$(aws cloudformation describe-stacks \
                        --stack-name raise13-ec2 \
                        --query 'Stacks[0].Outputs[?OutputKey==`InstancePublicIp`].OutputValue' \
                        --output text)
                      echo "export INSTANCE_PUBLIC_IP=$PUBLIC_IP" >> $BASH_ENV
            - run:
                  name: Retrieve RDS ENDPOINT of RDS
                  command: |
                      RDS_ENDPOINT=$(aws cloudformation describe-stacks \
                        --stack-name raise13-rds \
                        --query 'Stacks[0].Outputs[?OutputKey==`RDSInstanceEndpoint`].OutputValue' \
                        --output text)
                      echo "export RDS_ENDPOINT=$RDS_ENDPOINT" >> $BASH_ENV
            - run:
                  name: Retrieve ALB DNS Name
                  command: |
                      ALB_DNS_NAME=$(aws cloudformation describe-stacks \
                        --stack-name raise13-alb \
                        --query 'Stacks[0].Outputs[?OutputKey==`AlbDnsName`].OutputValue' \
                        --output text)
                      echo "export ALB_DNS_NAME=$ALB_DNS_NAME" >> $BASH_ENV
            - run:
                  name: Retrieve S3 Bucket Name
                  command: |
                      S3_BUCKET_NAME=$(aws cloudformation describe-stacks \
                        --stack-name raise13-s3 \
                        --query 'Stacks[0].Outputs[?OutputKey==`S3BucketName`].OutputValue' \
                        --output text)
                      echo "export S3_BUCKET_NAME=$S3_BUCKET_NAME" >> $BASH_ENV
            - run:
                  name: Retrieve Key Pair ID and Get Secret Key from SSM
                  command: |
                      KEY_PAIR_ID=$(aws ec2 describe-key-pairs --filters Name=key-name,Values=$KEY_NAME --query "KeyPairs[*].KeyPairId" --output text)
                      aws ssm get-parameter --name "/ec2/keypair/$KEY_PAIR_ID" --with-decryption --query "Parameter.Value" --output text > new-key-pair.pem
                      chmod 400 new-key-pair.pem
            - run:
                  name: Install Ansible and AWS dependencies
                  command: pip install ansible boto3 botocore
            - run:
                  name: Update Ansible Inventory
                  command: |
                      echo "[your_target_host_or_group]" > ansible/inventory.ini
                      echo "$INSTANCE_PUBLIC_IP ansible_ssh_user=ec2-user" >> ansible/inventory.ini
                      echo "[your_target_host_or_group:vars]" >> ansible/inventory.ini
                      echo "rds_endpoint=$RDS_ENDPOINT" >> ansible/inventory.ini
                      echo "alb_dns_name=$ALB_DNS_NAME" >> ansible/inventory.ini
                      echo "s3_buket_name=$S3_BUCKET_NAME" >> ansible/inventory.ini
                      echo "region=$REGION" >> ansible/inventory.ini
                      echo "ruby_version=3.1.2" >> ansible/inventory.ini
                      echo "bundler_version=2.3.14" >> ansible/inventory.ini
                      echo "nvm_version=0.39.5" >> ansible/inventory.ini
                      echo "nodejs_version=17.9.1" >> ansible/inventory.ini
            - run:
                  name: Run Ansible Playbook
                  command: |
                      export ANSIBLE_SSH_ARGS='-o StrictHostKeyChecking=no -i new-key-pair.pem'
                      ansible-playbook -i ansible/inventory.ini ansible/picture_upload_play.yml
            - run:
                  name: Setup SSH Config for Serverspec
                  command: |
                      mkdir -p ~/.ssh
                      cat \<< EOS >  ~/.ssh/config
                      Host raise13-ec2
                        Hostname $INSTANCE_PUBLIC_IP
                        User ec2-user
                        IdentityFile /home/circleci/project/new-key-pair.pem
                        StrictHostKeyChecking no
                        IdentitiesOnly yes
                      EOS
            - run:
                  name: install serverspec
                  command: |
                      sudo apt-get update
                      sudo apt-get install -y ruby ruby-dev build-essential
                      sudo gem install serverspec
                      sudo gem install ed25519
                      sudo gem install bcrypt_pbkdf
            - run:
                  name: run serverspec
                  command: |
                      cd serverspec
                      rake spec:raise13-ec2

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
            - run-ansible:
                  requires:
                      - deploy-s3
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
### `--capabilities CAPABILITY_NAMED_IAM`
CloudFormationテンプレートでIAMリソースを作成するために必要なもの。
### `no_output_timeout: 30m`
CircleCIは`.circleci/config.yml`で出力がない状態が10分を経過するとタイムアウトしてしまうので、その期限を30分に設定している。RDSは作成されるまでの時間が長いため10分ではタイムアウトする可能性がある。
### `$(aws cloudformation describe-stacks \`

### ```--query 'Stacks[0].Outputs[?OutputKey==`RegionName`].OutputValue' \```
対象のCloudFormaionテンプレートの`Outputs`で記述した論理IDを指定する。例えば、論理IDが`RegionName`であったら`?OutputKey==`RegionName``のように指定する。
### `KEY_PAIR_ID=$(aws ec2 describe-key-pairs --filters Name=key-name,Values=$KEY_NAME --query "KeyPairs[*].KeyPairId" --output text)`
`BASH_ENV`に保存した`$KEY_NAME`を`Values=$KEY_NAME`のようにして指定する。
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

# CircleCIからEC2にSSH接続するための準備
1. EC2のClodFormationテンプレートで`0.0.0.0/0`からのSSH(22)を許可する。※SSH(22)の`0.0.0.0/0`の許可はセキュリティリスクが高いためCircleCIのIPアドレスからのSSHを許可するのが望ましいが、今回は学習用ということもあり、あまりお金をかけられなかったのですべてを許可した。
2. Windowsのエクスプローラーから`.ssh`ディレクトリ内に`id_ed25519`や`id_ed25519.pub`といったファイルが存在しない時はコマンドプロンプトで`ssh-keygen -t ed25519 -C "your_email@example.com"`を実行する。
3. マネジメントコンソール上のEC2のページの「キーペア」をクリックして`id_ed25519.pub`の内容を登録していれば8からの手順を行い、登録していなければ引き続き以下の手順を行う。
4. マネジメントコンソール上のEC2のページの「キーペア」→「アクアション」→「キーペアをインポート」をクリックする。
5. 任意の名前を記述し、閲覧をクリックするとファイルをアップロードできるので`id_ed25519.pub`ファイルを選択する。
6. 一番下までスクロールして「キーペアをインポート」をクリックする。
7. EC2を作成する際にここで作成したキーペアを選択する
8. GitHubの右上のアイコンから「Settings」→「SSH And GPG Keys」をクリックする。「SSH keys」の項目で`id_ed25519.pub`を登録していれば完了となり、登録していなければ引き続き以下の手順を行う。
9. 「New SSH key」をクリックする。
10. 「Title」は任意の名前を記述し、「Key」には`id_ed25519.pub`のファイルに書かれている内容をすべてコピーして貼り付けて、「Add SSH key」をクリックする。
11. CircleCIの左側のサイドバーの「Projects」から対象のプロジェクトの「Set UP Project」を「Unfollow Project」の状態にする。
12. 対象のプロジェクトにある「・・・」→「Project Settings」をクリックする。
13. 左側のサイドバーの「SSH Keys」をクリックする。
14. 一番下までスクロールするして「Add SSH Key」をクリックする。
15. 「Private Key」に`id_ed25519`のファイルに書かれている内容をすべてコピーして貼り付けて、「Add SSH Key」をクリックする。

# CircleCIでトークンを発行
1. CircleCIの対象のプロジェクトの`・・・`から「Project Settings」をクリックする。
2. 左のサイドバーの「API Permissions」→「Add an API Token」をクリックする。
3. 「Scope」には「Read Only」を選択し、「Label」ではこのトークンが何の目的で作成したのかが後から見てわかるような名前を記入する。
4. 「Add API Token」をクリックするとトークンが表示されるのでコピーして安全な場所に保管する。

# 
