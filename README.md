# ETLハンズオン準備（リハ用）

## 概要
AWS Lambdaを使用して列車情報、気象情報をオープンデータAPIより取得し、DynamoDBへデータを格納します。
格納されたデータに対して、AWS Glueを使用してDynamoDBテーブルをクロールし、Amazon S3にデータを抽出し、Amazon AthenaでSQLクエリを使用して分析を実行する体験をします。

## 大まかな流れ
事前準備  
　東京公共交通オープンデータのアカウント登録  
　※APIアクセスに必要なコンシューマキーの取得が必要
　　https://developer-tokyochallenge.odpt.org/users/sign_up

### 1. AWSアカウントの確認とコンソールへのログイン
https://console.aws.amazon.com/console/home?nc2=h_ct&src=header-signin

### 2. IAMロールの作成
　→Lambda、DynamoDB、S3、Glueそれぞれについてのアクセスポリシーをアタッチしたロールを作成  
　※セキュリティは考慮しないので、ユーザーは作成せず、AWSアカウントに基本FullAccessを付けていく  
ロール名：role-jdmchandson  
アタッチするポリシーは下記の4つ  
AWSLambdaFullAccess  
AmazonDynamoDBFullAccess  
AmazonS3FullAccess  
AWSGlueServiceRole  
※以降のGlueの設定時に既存ロールを選すると、DynamoDBテーブルへのクロール用ポリシーが追加される

### 3. 列車情報取得用Lambdaファンクションの作成
* GitHubからダウンロード
https://github.com/mimopa/traindata-to-dynamodb.git
* 展開されたフォルダの中身をzipファイルに圧縮  
→ディレクトリは含まないように注意
$ zip -r traindata-to-dynamodb.zip *
* Lambdaコンソールからzipファイルをアップロード
* Lambdaの環境変数に東京公共交通オープンデータのアクセスキーを設定する  
環境変数：「CONSHUMER_KEY」に設定

### 4. CloudWatchイベントの作成  
→5分間隔での実行
* CloudWatchコンソールからルールの作成  
  clone式で5分ごと：0/5 * * * ? *  
  ターゲット関数：traindata-to-dynamodb

### 5. 気象情報取得用Lamnbaファンクションの作成
* Githubからダウンロード
https://github.com/mimopa/otenki_scraping_to_dynamodb.git  
※「headless-chromium」が100mbほどあるので、GitHubｈには上げられない。。。  
* GoogleドライブからLambda関数用のzipファイルをダウンロードする
  https://drive.google.com/drive/folders/1UwgFXE45J0GTd4qrogzG5ZD4iJzdXOED?usp=sharing

### 6. CloudWatchイベントの作成
→1時間間隔での実行  
  clone式で１時間ごと：0 0/1 * * ? *  
  ターゲット関数：otenki_scraping_to_dynamodb

### 7. AWSGlueのクローラ作成
※参考
https://aws.amazon.com/jp/blogs/news/simplify-amazon-dynamodb-data-extraction-and-analysis-by-using-aws-glue-and-amazon-athena/
* データカタログ、データ格納用のS3バケットを作成  
  バケット名：publictransportdata  
  アクセス権限：パブリック  
* DynamoDBからスキーマを検出し、AWS Glue Data Catalogにメタデータを設定するクローラを作成  
  クローラー名：traindata_for_dynamodb  
  データストア：DnynamoDB  
  テーブル名：train_catalog  
  IAMロール：role-jdmchandson  
  ※既存のIAMロールを選択すると、DynamoDBをクロールするポリシーが追加される  
  →AWSGlueServiceRole-default  
  頻度：オンデマンドで実行 ※ハンズオンでは手動実行  
* DynamoDBテーブルからデータを抽出し、S3に格納するETLジョブを作成  
  ジョブの名前：train_for_dynamodb  
  IAMロール：AWSGlueServiceRole-default  
  Type：Spark  
  データソース：train_catalog  
  データターゲットでテーブルを作成する  
  データストア：AmazonS3  
  形式：JSON  
  ターゲットパス：s3://publictransportdata/train  
* S3のスキーマを検出し、AWS Glue Data Catalogにメタデータを設定するクローラを作成  
  クローラ名：traindata_for_S3  
  データストア：S3  
  クロールするデータの場所：指定されたパス  
  インクルードパス：s3://publictransportdata/train  
  出力を設定では、データベースを追加  
  データベース：traindata_for_S3  
* データカタログに登録されたメタデータテーブルに対してAthenaでSQLを発行してデータを確認  
※Glueコンソールのテーブルから確認可能  
→データカタログ用のテーブルは確認メニューがグレーアウトされている。  
### 8. 外部からの接続  
Athenaで参照可能となったデータカタログを、JDBCで外部から接続、クエリの実行を可能とする。  
※必要なJDBCドライバをダウンロードし、ハンズオン用資材として保存しておく。  
  ドライバ：  
  クラス名：com.simba.athena.jdbc.Driver  
  URL：jdbc:awsathena://athena.ap-northeast-1.amazonaws.com:443  
  Username:AWSアクセスキー  
  Password：AWSシークレットキー  
