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

# CircleCIの`.circleci/config.yml`
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
`$BASH_ENV`というCircleCIの特別なファイルに保存することで`Outputs`によって出力された値をCircleCIで参照することができる。`$BASH_ENV`からの参照は異なるステップ間なら可能だが、異なるジョブ間では参照できないので再度同じように`Outputs`によって出力された値を`$BASH_ENV`に保存する処理を書かなければならない。
### ```--query 'Stacks[0].Outputs[?OutputKey==`RegionName`].OutputValue' \```
対象のCloudFormaionテンプレートの`Outputs`で記述した論理IDを指定する。例えば、論理IDが`RegionName`であったら`?OutputKey==`RegionName``のように指定する。
### `KEY_PAIR_ID=$(aws ec2 describe-key-pairs --filters Name=key-name,Values=$KEY_NAME --query "KeyPairs[*].KeyPairId" --output text)`
`$BASH_ENV`に保存した`$KEY_NAME`を`Values=$KEY_NAME`のようにして指定する。
### `echo "[your_target_host_or_group]" > ansible/inventory.ini`
`[your_target_host_or_group]`はE2にSSH接続するときに必要な情報を記述するための項目で、Ansibleのインベントリファイルに記述する。
### `echo "[your_target_host_or_group:vars]" >> ansible/inventory.ini`
`[your_target_host_or_group:vars]`はAnsibleプレイブック内で使用する変数を定義するための項目。
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

# Ansibleプレイブック
```yaml

---
- name: Install git on target hosts
  hosts: your_target_host_or_group
  become: yes

  tasks:
      - name: Install git
        yum:
            name: git
            state: present

- name: Remove existing directory and Clone the repository
  hosts: your_target_host_or_group
  tasks:
      - name: Remove existing directory
        file:
            path: "/home/ec2-user/raisetech-live8-sample-app"
            state: absent

      - name: Clone the repository
        git:
            repo: "https://github.com/yuta-ushijima/raisetech-live8-sample-app.git"
            dest: "/home/ec2-user/raisetech-live8-sample-app"

- name: Yum Update Playbook
  hosts: your_target_host_or_group
  become: yes

  tasks:
      - name: Run yum update
        yum:
            name: "*"
            state: latest

- name: Install packages using yum
  hosts: your_target_host_or_group
  become: yes
  tasks:
      - name: Install curl, gpg, gcc, gcc-c++, make
        yum:
            name:
                - curl
                - gpg
                - gcc
                - gcc-c++
                - make
            state: present

- name: Run curl and gpg2 command
  hosts: your_target_host_or_group
  tasks:
      - name: Execute curl command and pipe to gpg2
        shell: curl -sSL https://rvm.io/mpapis.asc | gpg2 --import -

- name: Git Install Play
  hosts: your_target_host_or_group
  become: yes
  tasks:
      - name: GPG Key Import
        shell: curl -sSL https://rvm.io/pkuczynski.asc | gpg2 --import -

- name: Install RVM and Ruby
  hosts: your_target_host_or_group
  tasks:
      - name: Install GPG keys for RVM
        shell: |
            gpg2 --keyserver hkp://keyserver.ubuntu.com --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3 7D2BAF1CF37B13E2069D6956105BD0E739499BDB
        args:
            executable: /bin/bash
        ignore_errors: yes

      - name: Install RVM via curl
        shell: |
            curl -sSL https://get.rvm.io | bash -s stable
        args:
            executable: /bin/bash

      - name: Source RVM in .bashrc
        lineinfile:
            path: ~/.bashrc
            line: "source ~/.rvm/scripts/rvm"

- hosts: your_target_host_or_group
  tasks:
      - name: Check if Ruby {{ ruby_version }} is already installed
        shell: source ~/.rvm/scripts/rvm && rvm list | grep 'ruby-{{ ruby_version }}'
        args:
            executable: /bin/bash
        register: ruby_installed
        ignore_errors: yes

      - name: Source RVM script and install Ruby {{ ruby_version }}
        shell: source ~/.rvm/scripts/rvm && rvm install ruby-{{ ruby_version }}
        args:
            executable: /bin/bash
        when: ruby_installed.rc != 0

- name: Install bundler
  hosts: your_target_host_or_group
  tasks:
      - name: Install bundler version {{ bundler_version }}
        shell: source ~/.rvm/scripts/rvm && gem install bundler -v {{ bundler_version }}
        args:
            executable: /bin/bash

- name: Install mysql-devel
  hosts: your_target_host_or_group
  become: yes

  tasks:
      - name: Install mysql-devel package
        yum:
            name: mysql-devel
            state: latest

- name: Bundle install
  hosts: your_target_host_or_group
  tasks:
      - name: bundle install
        command:
            cmd: bundle install
            chdir: /home/ec2-user/raisetech-live8-sample-app

- name: Create MySQL database
  hosts: your_target_host_or_group
  vars:
      ansible_python_interpreter: /usr/bin/python3
  tasks:
      - name: Install pip3 if not exists
        become: true
        package:
            name: python3-pip
            state: present

      - name: Install PyMySQL
        become: true
        pip:
            name: PyMySQL

      - name: Create database
        mysql_db:
            name: "Raise13"
            state: present
            login_host: "{{ rds_endpoint }}"
            login_user: "admin"
            login_password: "{{ lookup('aws_ssm', 'rds_master_password', region=region) }}"

- name: Deploy database.yml from template
  hosts: your_target_host_or_group
  vars:
      database_table_name: "Raise13"
      database_username: "admin"
      database_password: "{{ lookup('aws_ssm', 'rds_master_password', region=region) }}"
      database_endpoint: "{{ rds_endpoint }}"
      database_port: 3306
  tasks:
      - name: Create database.yml from template
        template:
            src: templates/database.yml.j2
            dest: /home/ec2-user/raisetech-live8-sample-app/config/database.yml
            owner: ec2-user
            group: ec2-user
            mode: "0664"

- name: Execute curl command in /home/ec2-user/raisetech-live8-sample-app
  hosts: your_target_host_or_group
  tasks:
      - name: Execute curl command to install nvm
        shell: |
            cd /home/ec2-user/raisetech-live8-sample-app
            curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v{{ nvm_version }}/install.sh | bash
        args:
            executable: /bin/bash

- name: Set persistent environment variable
  hosts: your_target_host_or_group
  tasks:
      - name: Add export NVM_DIR="$HOME/.nvm" to .bashrc
        lineinfile:
            path: /home/ec2-user/.bashrc
            line: 'export NVM_DIR="$HOME/.nvm"'
            state: present

- name: Execute command in /home/ec2-user/raisetech-live8-sample-app
  hosts: your_target_host_or_group
  tasks:
      - name: Execute nvm command
        shell:
            cmd: '[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"'
            chdir: /home/ec2-user/raisetech-live8-sample-app

- hosts: your_target_host_or_group
  tasks:
      - name: Execute nvm install nodejs in specific directory
        shell: |
            . ~/.nvm/nvm.sh
            nvm install {{ nodejs_version }}
        args:
            chdir: /home/ec2-user/raisetech-live8-sample-app

- name: A playbook with global environment variables
  hosts: your_target_host_or_group
  become: yes
  tasks:
      - name: Install yarn globally using nvm
        become_user: ec2-user
        shell: |
            . ~/.nvm/nvm.sh
            npm install -g yarn
        environment:
            PATH: "/home/ec2-user/.nvm/versions/node/v{{ nodejs_version }}/bin:{{ ansible_env.PATH }}"

- name: Install ImageMagick on target hosts
  hosts: your_target_host_or_group
  become: yes

  tasks:
      - name: Install ImageMagick
        yum:
            name: ImageMagick
            state: present

- name: Install nginx on EC2 instances
  hosts: your_target_host_or_group
  become: yes
  tasks:
      - name: Execute the command to install nginx
        command:
            cmd: amazon-linux-extras install nginx1 -y
            chdir: /home/ec2-user/raisetech-live8-sample-app

- name: Run bundle install in /home/ec2-user/raisetech-live8-sample-app
  hosts: your_target_host_or_group
  tasks:
      - name: Run bundle install
        command: bundle install
        args:
            chdir: /home/ec2-user/raisetech-live8-sample-app
        become: yes
        become_user: ec2-user

- name: Change unicorn.rb
  hosts: your_target_host_or_group
  tasks:
      - name: Create unicorn.rb
        template:
            src: templates/unicorn.rb.j2
            dest: /home/ec2-user/raisetech-live8-sample-app/config/unicorn.rb
            owner: ec2-user
            group: ec2-user
            mode: "0664"

- name: Change nginx.conf
  hosts: your_target_host_or_group
  become: yes
  tasks:
      - name: Create nginx.conf
        template:
            src: templates/nginx.conf.j2
            dest: /etc/nginx/nginx.conf
            owner: root
            group: root
            mode: "0644"

- name: Change strage.yml
  hosts: your_target_host_or_group
  become: yes
  vars:
      S3_BUKET_NAME: "{{ s3_buket_name }}"
      REGION: "{{ region }}"
  tasks:
      - name: Create strage.yml
        template:
            src: templates/storage.yml.j2
            dest: /home/ec2-user/raisetech-live8-sample-app/config/storage.yml
            owner: ec2-user
            group: ec2-user
            mode: "0664"

- name: Change development.rb
  hosts: your_target_host_or_group
  become: yes
  vars:
      ALB_DNS_NAME: "{{ alb_dns_name }}"
  tasks:
      - name: Create development.rb
        template:
            src: templates/development.rb.j2
            dest: /home/ec2-user/raisetech-live8-sample-app/config/environments/development.rb
            owner: ec2-user
            group: ec2-user
            mode: "0664"

- name: Run RAILS_ENV=development bundle exec rake assets:precompile
  hosts: your_target_host_or_group
  tasks:
      - name: Execute rake assets:precompile
        shell:
            cmd: RAILS_ENV=development bundle exec rake assets:precompile
            chdir: /home/ec2-user/raisetech-live8-sample-app

- name: Rails database migration
  hosts: your_target_host_or_group
  become: yes
  become_user: ec2-user
  tasks:
      - name: Check if migrations are needed
        shell: >
            cd /home/ec2-user/raisetech-live8-sample-app &&
            bin/rails runner "exit ActiveRecord::Base.connection.migration_context.needs_migration? ? 1 : 0"
        environment:
            RAILS_ENV: development
        args:
            executable: /bin/bash
        register: migrations_needed
        ignore_errors: yes

      - name: db:migrate run if needed
        shell: cd /home/ec2-user/raisetech-live8-sample-app && bin/rails db:migrate RAILS_ENV=development
        environment:
            RAILS_ENV: development
        args:
            executable: /bin/bash
        when: migrations_needed.rc == 1

- name: Start nginx service
  hosts: your_target_host_or_group
  tasks:
      - name: Ensure nginx is started
        become: yes
        command:
            cmd: systemctl start nginx
            chdir: /home/ec2-user/raisetech-live8-sample-app

- name: Run Unicorn command on remote server
  hosts: your_target_host_or_group
  become: yes
  become_user: ec2-user
  tasks:
      - name: Execute bundle exec unicorn command
        shell: cd /home/ec2-user/raisetech-live8-sample-app && bundle exec unicorn -c /home/ec2-user/raisetech-live8-sample-app/config/unicorn.rb -E development -D
```
## 解説
### `hosts`
インベントリファイルに記載しているEC2にSSH接続するための情報が代入された変数を`hosts`で指定する。インベントリファイルには以下のように記載されている。

```
[your_target_host_or_group]
[EC2のパブリックIPv4アドレス] ansible_ssh_user=ec2-user
```

### `become`
`yes`を指定するとroot権限で処理が実行される。`no`を指定すると明示的にインベントリファイルの`ansible_ssh_user`に記載したユーザー(今回は`ec2-user`)で処理が実行される。プレイブックに`become`を記載しないのと`become: no`を記載するのはどちらも`ansible_ssh_user`に記載したユーザーで実行されることは同じだが`become: no`と記載することでコードの可読性を向上させたり、プレイブックがデフォルトでroot権限で実行されるようになっていた場合に`ansible_ssh_user`に記載したユーザーに上書きすることができる。
### `state`
`present`を指定するとパッケージがインストールされていないときのみインストールを行うので常に同じバージョンでパッケージを使用できる。`absent`を指定するとパッケージのアンインストールが行われる。`latest`を指定するとパッケージがインストールされていなければインストールし、既にインストールされている場合でも最新のバージョンにアップグレードする。
### `dest`
どこにデータを保存するかを指定する。
### `chdir`
どこでそのコマンドを実行するかを指定する。プレイブックに`chdir`を記載しない場合、対象のサーバーのホームディレクトリ(今回は`/home/ec2-user`ディレクトリ)でコマンドが実行される。
### `shell`
シェルの機能（パイプライン |、リダイレクト >, <、環境変数など）を使ってコマンドを実行するときに使うもの。
### `command`
シェルの機能（パイプライン |、リダイレクト >, <、環境変数など）を使わない単純なコマンドを実行するときに使うもの。`shell`ではなく`command`を使うことで互換性やセキュリティリスクを低減させることができる。
### `shell: |`
複数のコマンドを連結させて実行したり、複数のコマンドを複数の行に記述する際に使うもの。
### `args`
`executable`や`chdir`を含む追加の設定を行う際に必要なもの。
### `cmd`
実行するコマンドを記述する。本来は`shell`または`command`で実行するコマンドを記述するが`cmd`を使う場合は`cmd`に記述する。`cmd`を使うことで`executable`または`chdir`を使う際に必要となる`args`セクションが不要になる。
### `executable`
`shell`モジュールを実行する際に使用するシェルの種類を指定するセクション。`executable`を使わずに`shell`モジュールを実行すると`/bin/sh`というデフォルトのシェルを使って実行される。
### `ignore_errors`
`yes`を指定するとタスクでエラーが発生しても、そのタスクのエラーを無視して後続のタスクを処理する。
### `lineinfile`
コンピューターが起動する際に読み込むファイルに設定を書き足すためのもの。今回は`path`に設定されている`.bashrc`というファイルに設定が書き足されている。
### `register`
コマンドの実行結果を保存する役割を持つ。上記では実行結果が保存された`ruby_installed`という変数を使うことで、後続のタスクでもコマンドの実行結果を使用できる。
### `when: ruby_installed.rc != 0`
`when`を使うことで条件分岐が行えるようになり、`rc`にはコマンドの実行が成功したか失敗したかを表す数字が格納されている。コマンドが成功した場合は0、失敗した場合は0以外の数字が格納される。
### `ansible_python_interpreter: /usr/bin/python3`
AnsibleがどのPythonのバージョンで処理を行うのかを指示してあげるものであり、`ansible_python_interpreter: /usr/bin/python3`を指示したタスクからプレイブックのコードの最後まで反映される。
### `become: true`
`become: yes`と同じ意味。`become`にはtrue、false、yes、noのいずれかを選択する。
### ```login_password: "{{ lookup('aws_ssm', 'rds_master_password', region=region) }}"```

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
