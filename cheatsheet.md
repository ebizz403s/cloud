# Bash



# Cloud Command Workflows

## Google Cloud Platform

### Projects

To see projects:
```bash
gcloud config list
gcloud config get-value project
gcloud config configurations list
```
To set projects:
```bash
gcloud config set project YOUR_PROJECT_ID
```

### Buckets

To see buckets:
```bash
gcloud storage buckets list --project "$(gcloud config get-value project)"
gsutil ls
gcloud storage buckets describe gs://YOUR_BUCKET_NAME
```

List buckets with access:
```bash
gsutil ls
```

List contents of a buckets:
```bash
gsutil ls gs://ita-techcrunch-rss-jsonl-bucket/
```

List objects under a prefix
```bash
gsutil ls gs://ita-techcrunch-rss-jsonl-bucket/rss-landing/source=techcrunch/`
```

Recursive listing (everything under a path)
```bash
gsutil ls -r gs://ita-techcrunch-rss-jsonl-bucket/some/prefix/**
```

Long format (size + timestamp)
```bash
gsutil ls -l gs://ita-techcrunch-rss-jsonl-bucket/some/prefix/**
```

List only “directories” / prefixes (not individual objects)
```bash
gsutil ls -d gs://ita-techcrunch-rss-jsonl-bucket/some/prefix/*
```

### Create Bucket

```bash
gcloud config set project YOUR_PROJECT_ID
gcloud storage buckets create gs://ita-wsj-rss-jsonl-bucket \
  --location=us-central1 \
  --uniform-bucket-level-access
```

### Storage Utility

`gsutil` = Google Storage utility

Sync
```bash
gsutil rsync -r ./local/ gs://my-bucket/local/
```

Remove
```bash
gsutil rm gs://my-bucket/path/file.jsonl
```

Copy
```bash
gsutil cp file.txt gs://my-bucket/
```

### Service Accounts

A service account is basically a robot identity in Google Cloud. Humans log in with user accounts (your email). Cloud Functions, Cloud Run, schedulers, VMs, pipelines, etc. need a way to authenticate too—so they use service accounts.

You typically have two service accounts:

#### Runtime service account (Function’s identity)

Your function runs “as” this account. If it needs to write JSONL files into your bucket, you grant it something like:
`roles/storage.objectAdmin` on that bucket.

#### Caller service account (Scheduler’s identity)

Cloud Scheduler calls your HTTP function using OIDC (a signed identity token). That service account needs permission to invoke the function:
`roles/run.invoker` (for Gen2 functions, because they run on Cloud Run).

Permissions ?
```bash
gsutil iam ch ...
gsutil setmeta ...
```




### Pipeline 02: Functions

See projects (examples: fetch_prnewswire):
```bash
gcloud functions list
gcloud functions list --gen2 --regions=us-central1
```

Describe function, get values:
```bash
gcloud functions describe fetch_prnewswire
gcloud functions describe fetch_prnewswire --gen2 --region us-central1
gcloud functions describe fetch_prnewswire --gen2 --region us-central1 --format="value(serviceConfig.uri)"

```

### Enable APIs
```bash
gcloud services enable \
  cloudfunctions.googleapis.com \
  run.googleapis.com \
  cloudscheduler.googleapis.com \
  cloudbuild.googleapis.com \
  artifactregistry.googleapis.com \
  storage.googleapis.com \
  iam.googleapis.com
```


## Amazon Web Services

Storage
`s3://` for cloud AWS S3 storage
