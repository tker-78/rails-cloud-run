# Deployment of Rails applications to Google Cloud Run


APIの有効化
```bash
$ gcloud services enable run.googleapis.com sql-component.googleapis.com \
  cloudbuild.googleapis.com secretmanager.googleapis.com compute.googleapis.com \
  sqladmin.googleapis.com
```

sqlインスタンスの作成
```bash
$ gcloud sql instances create rails-instance \
  --database-version POSTGRES_12 \
  --tier db-f1-micro \
  --region asia-northeast1
```

データベースの作成
```bash
$ gcloud sql databases create rails-database \
  --instance rails-instance
```

データベースのユーザーパスワードを作成
```bash
$ cat /dev/urandom | LC_ALL=C tr -dc '[:alpha:]'| fold -w 50 | head -n1 > dbpassword
```

```bash
$ gcloud sql users create rails-user \
  --instance rails-instance \
  --password=$(cat dbpassword)
```

Cloud Storageのバケットの作成
```bash
$ gsutil mb -l asia-northeast1 gs://metal-figure-412823-media
```

```bash
$ gsutil iam ch allUsers:objectViewer gs://metal-figure-412823-media
```


Secret Manager

```bash
$ EDITOR=vim bin/rails credentials/edit
```

```bash
$ gcloud secrets create rails-secret --data-file config/master.key
```

```bash
$ gcloud secrets describe rails-secret
```

プロジェクトへのアクセス権を与える
```bash
$ gcloud secrets add-iam-policy-binding rails-secret \
  --member serviceAccount:482406439394-compute@developer.gserviceaccount.com \
  --role roles/secretmanager.secretAccessor
```

```bash
$ gcloud secrets add-iam-policy-binding rails-secret \
  --member serviceAccount:482406439394@cloudbuild.gserviceaccount.com \
  --role roles/secretmanager.secretAccessor
```

```bash
$ gcloud projects add-iam-policy-binding metal-figure-412823 \
    --member serviceAccount:482406439394@cloudbuild.gserviceaccount.com \
    --role roles/cloudsql.client
```

ビルド
```bash
$ gcloud builds submit --config cloudbuild.yaml 
```

デプロイ
```bash
$ gcloud run deploy rails-cloud-run \
  --platform managed \
  --region asia-northeast1 \
  --image gcr.io/metal-figure-412823/rails-cloud-run \
  --add-cloudsql-instances metal-figure-412823:asia-northeast1:rails-instance \
  --allow-unauthenticated
```

以上でうまくいった。
ポイントは、`cloudbuild.yaml`を正しく記述すること。
