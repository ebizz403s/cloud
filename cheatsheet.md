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

Permissions ?
```bash
gsutil iam ch ...
gsutil setmeta ...
```




### Pipeline 02: Functions

See projects
```bash
gcloud functions list --gen2 --regions=us-central1
gcloud functions list --regions=us-central1
```




## Amazon Web Services

Storage
`s3://` for cloud AWS S3 storage
