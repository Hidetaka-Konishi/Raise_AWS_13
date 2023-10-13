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
3. 
