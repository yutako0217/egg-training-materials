# EGG ハンズオン #5 Cloud Run 編

## **Google Cloud プロジェクトの選択**

ハンズオンを行う Google Cloud プロジェクトを選択して **Start/開始** をクリックしてください。

<walkthrough-project-setup>
</walkthrough-project-setup>

<walkthrough-watcher-constant key="region" value="asia-northeast1"></walkthrough-watcher-constant>

## **ハンズオンの事前準備**
後続の手順で利用しますので、環境変数の設定を行います。
任意のリポジトリ名を、設定してください。
```shell
export REPOSITORY_NAME=""
```


先程の手順で利用した環境変数も利用します。
Cloud Shell の接続が切れた場合はこちらも設定してください。
```shell
export PROJECT_ID="{{project-id}}"
export INSTANCE_ID=""
export DATABASE_ID=""
export SA_NAME=""
export SA_KEY_NAME=""
```

APIの有効化
```shell
artifactregistry.googleapis.com
run.googleapis.com
```

## **2. コンテナイメージの作成と Cloud Run へのデプロイ方法**
### **1. リポジトリを作成（Artifact Registry）**

```shell
gcloud artifacts repositories create ${REPOSITORY_NAME} --repository-format=docker --location={{region}} --description="Docker repository for Cloud Run hands-on"
```

リポジトリにコンテナイメージを Push するための権限を付与
```shell
gcloud auth configure-docker asia-northeast1-docker.pkg.dev
```

[コンソールでも](https://console.cloud.google.com/artifacts)から、リポジトリの確認が可能です。

## **2-1. Dockerfile を使用してローカルでコンテナを作成、Artifact Registry 経由でデプロイする**

### **1. コンテナイメージを作成**

```shell
# cloneするディレクトリを指定する。とりあえずHome直下
cd ~/spanner-sqlalchemy-demo 
docker build -t asia-northeast1-docker.pkg.dev/${PROJECT_ID}/${REPOSITORY_NAME}/spanner-sqlalchemy-demo:1.0.0 .
```

### **2. コンテナイメージを、Artifact Registry に Push**
```shell
# cloneするディレクトリを指定する。とりあえずHome直下
cd spanner-sqlalchemy-demo 
docker push asia-northeast1-docker.pkg.dev/${PROJECT_ID}/${REPOSITORY_NAME}/spanner-sqlalchemy-demo:1.0.0
```

### **3. Cloud Runへデプロイ**
```shell
gcloud run deploy spanner-sqlalchemy-demo \
--image asia-northeast1-docker.pkg.dev/${PROJECT_ID}/${REPOSITORY_NAME}/spanner-sqlalchemy-demo:1.0.0 \
--allow-unauthenticated \
--set-env-vars=PROJECT_ID=${PROJECT_ID},INSTANCE_ID=${INSTANCE_ID},DATABASE_ID=${DATABASE_ID} \
--service-account=${SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com \
--region=asia-northeast1
```

### **4. サービスの動作確認**

```bash
APP_URL=$(gcloud run services describe spanner-sqlalchemy-demo --format json | jq -r '.status.address.url')
curl -s APP_URL
```

### **5. サービスの削除**

```bash
gcloud run services delete spanner-sqlalchemy-demo --quiet
```

## **2-2. Buildpacks、Cloud Buildを使用してデプロイする**

### **1. Dockerfile の削除（移動）**

Dockerfile 無しでデプロイできることを確かめるために、Dockerfile を退避します。

```bash
mv Dockerfile ./
```

**ヒント**: Buildpacks というソフトウェアを使い、Dockerfile 無しでのデプロイを実現しています。
詳細は[こちら](https://cloud.google.com/blog/ja/products/containers-kubernetes/google-cloud-now-supports-buildpacks)を参照してください。

### **2. 一括でデプロイ**

```bash
gcloud run deploy spanner-sqlalchemy-demo --source ./ --allow-unauthenticated
```

## **3. トラフィックコントロール**

