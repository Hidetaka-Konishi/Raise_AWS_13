# CircleCIでスタックをデプロイするリージョンを環境変数として設定する手順
1. CircleCIにログインする。
2. 左サイドバーの「Projects」をクリックする。
3. 対象のプロジェクトにある「・・・」をクリックする。
4. 「Project Settings」をクリックする。
5. 左サイドバーの「Environment Variables」→「Add Environment Variable」をクリックする。
6. 「Name」に`AWS_REGION`、「Value」に設定したいリージョン名を指定する。例：us-east-1
7. 「Add Enviroment Variable」をクリックする。
