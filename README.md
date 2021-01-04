# **Docker-ECR-Codebuild**

## **About**

gitレポジトリーの変更をトリガーにECRへのPUSH、およびCodebuildをする

## **Inside Package**
 * web nginx:alpine
 * app php
 * DBはAWSのRDSを使用

## **手順**

### **1. ECRレポジトリの作成**

* AWSマネコンで　ECR > リポジトリ > リポジトリを作成
```
 可視性設定　プライベート
 リポジトリ名　Laravel-app-ecs(例)
 リポジトリを作成
```

### **2. 「プッシュコマンドの表示」をクリックし、ポップアップの内容に従いローカルターミナルで以下**

* Docker クライアントを認証
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

* 確認
<br>AWSマネコンで ECR > リポジトリ > laravel-app-ecs（例）でweb, appの2つのイメージが保存されていることを確認する


### 3. CodeBuildの作成

* AWSマネコンで　デベロッパー用ツール > CodeBuild > ビルドプロジェクト > ビルドプロジェクトを作成
```
プロジェクトの設定
  プロジェクト名　laravel-app-ecs-build（例）

ソース
  ソースプロバイダ　Github
  リポジトリ　OAuth を使用して接続する　を選択
  Githubに接続　をクリック
  ポップアップで　確認　を選択
  GitHub リポジトリ　https://github.com/siwai0208/ecs-pipeline.git　を選択

環境
  環境イメージ　マネージド型イメージ
  オペレーティングシステム　Amazon Linux 2
  ランタイム　Standard
  イメージ　aws/codebuild/amazonlinux2-x86-64-standard:3.0
  特権付与　有効化する
  サービスロール　新しいサービスロールを選択
  追加設定　環境変数に以下の値を追加
    AWS_ACCOUNT_ID
    AWS_DEFAULT_REGION
    DOCKERHUB_USER
    DOCKERHUB_PASS
    IMAGE_REPO_NAME

ビルドプロジェクトを作成する
```

### 4. CodePieplineの作成

* AWSマネコンで　デベロッパー用ツール > CodePipeline > パイプライン > パイプラインを作成する
```
パイプラインの設定
  パイプライン名　laravel-app-ecs-pipeline（例）
  高度な設定　laravel-app-image（例）

ソースステージを追加する
  ソースプロバイダー　Github(バージョン2)
  接続　Githubに接続する
  接続名　food-app-ecs-pipeline
  GitHub アプリ　「3. CodeBuildの作成」で作成したアプリを選択し接続
  リポジトリ名　siwai0208/ecs-pipeline
  ブランチ名　main

ビルドステージを追加する
  プロバイダーを構築する　AWS CodeBuild
  プロジェクト名　laravel-app-ecs-build（3. CodeBuildの作成で作成）
  次に

デプロイ
  導入段階をスキップ
  パイプラインを作成する
```

### 5. CodePipelineのテスト

* パイプラインを作成後、自動でソースチェック→BuildがスタートするがBuildフェーズで失敗となる。

* AWSマネコン　IAM > ロール で「3. CodeBuildの作成」で作成した新しいサービスロールを選択し「AmazonEC2ContainerRegistryFullAccess」「AmazonS3FullAccess」をポリシーをアタッチ

* Codepipelineに戻り、Buildを再試行
