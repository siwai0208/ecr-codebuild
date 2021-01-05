# **Docker-ECR-Codebuild**

## **About**

gitレポジトリーの変更をトリガーにCodebuild→ECRへ更新されたイメージをPUSH

## **Inside Package**
 * docker-compose.yml
   <br>web nginx:alpine
   <br>app php-fpm
   <br>DBはAWSのRDSを使用
 * buildspec.yml
   <br>ECR login→build→tagづけ→ECRへpush

## **手順**

### **1. ECRレポジトリの作成**

* AWSマネコンで　ECR > リポジトリ > リポジトリを作成
```
 可視性設定　プライベート
 リポジトリ名　Laravel-app-ecs(例)
 リポジトリを作成
```

### **2. イメージの構築とPUSH**

* 「プッシュコマンドの表示」をクリックし、ポップアップの内容に従いローカルターミナルでDockerクライアント認証
```
aws ecr get-login-password --region ap-northeast-1 | docker login --username AWS --password-stdin xxxxxxxxxxxx.dkr.ecr.ap-northeast-1.amazonaws.com
...

Login Succeeded　←成功
```

* イメージを構築
```
docker-compose build
```

* イメージにタグ付け
docker images コマンドで ecs-pipeline__webと、ecs-pipeline__appの2つがビルドされていることを確認し、それぞれのイメージにタグ付けをする

```
docker images
 REPOSITORY        TAG 
 ecs-pipeline_web  latest
 ecs-pipeline_app  latest

docker tag ecs-pipeline_web:latest xxxxxxxxxxxx.dkr.ecr.ap-northeast-1.amazonaws.com/laravel-app-ecs:web
docker tag ecs-pipeline_app:latest xxxxxxxxxxxx.dkr.ecr.ap-northeast-1.amazonaws.com/laravel-app-ecs:app
```

* イメージをECRリポジトリにPUSH

```
docker push xxxxxxxxxxxx.dkr.ecr.ap-northeast-1.amazonaws.com/laravel-app-ecs:web
docker push xxxxxxxxxxxx.dkr.ecr.ap-northeast-1.amazonaws.com/laravel-app-ecs:app
```

* AWSマネコンで ECR > リポジトリ > laravel-app-ecs（例）でweb, appの2つのイメージが保存されていることを確認する


### **3. CodeBuildの作成**

* AWSマネコンで　デベロッパー用ツール > CodeBuild > ビルドプロジェクト > ビルドプロジェクトを作成
```
プロジェクトの設定
  プロジェクト名　laravel-app-ecr-build（例）

ソース
  ソースプロバイダ　Github
  リポジトリ　OAuth を使用して接続する　を選択
  Githubに接続　をクリック
  ポップアップで　確認　を選択
  GitHub リポジトリ　https://github.com/siwai0208/ecr-codebuild.git　を選択

環境
  環境イメージ　マネージド型イメージ
  オペレーティングシステム　Amazon Linux 2
  ランタイム　Standard
  イメージ　aws/codebuild/amazonlinux2-x86-64-standard:3.0
  イメージのバージョン  常に最新のイメージ... を選択
  特権付与　有効化する
  サービスロール　新しいサービスロールを選択
  追加設定　環境変数に以下の値を追加
    名前:値
    AWS_ACCOUNT_ID: YOUR-AWS-ACCOUNT-ID
    AWS_DEFAULT_REGION: YOUR-AWS-DEFAULT-REGION
    DOCKERHUB_USER: YOUR-DOCKERHUB-USER
    DOCKERHUB_PASS: YOUR-DOCKERHUB-PASS
    IMAGE_REPO_NAME: ECR-REPO-NAME

ビルドプロジェクトを作成する
```

### 4. CodePieplineの作成

* AWSマネコンで　デベロッパー用ツール > CodePipeline > パイプライン > パイプラインを作成する
```
パイプラインの設定
  パイプライン名　ecr-codebuild-pipeline（例）
  高度な設定　S3バケットを選択　laravel-app-image（例）

ソースステージを追加する
  ソースプロバイダー　Github(バージョン2)
  接続　Githubに接続する
  接続名　ecr-codebuild
  GitHub アプリ　新しいアプリをインストールする
  Repository access > Only select repositories > 
  リポジトリ名　siwai0208/ecr-codebuild
  ブランチ名　main

ビルドステージを追加する
  プロバイダーを構築する　AWS CodeBuild
  プロジェクト名　laravel-app-ecr-build（3. CodeBuildの作成で作成）
  次に

デプロイ
  導入段階をスキップ
  パイプラインを作成する
```

### 5. CodePipelineのテスト

* パイプラインを作成後、自動でソースチェック→BuildがスタートするがBuildフェーズで失敗となる。

* AWSマネコン　IAM > ロール で「3. CodeBuildの作成」で作成した新しいサービスロールを選択し「AmazonEC2ContainerRegistryFullAccess」「AmazonS3FullAccess」をポリシーをアタッチ

* Codepipelineに戻り、Buildを再試行→成功し、ECRのイメージが更新される

* 以降、gitレポジトリーの変更をトリガーにCodebuild→ECRへ更新されたイメージがPUSHされる

