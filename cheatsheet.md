# Bash

Set variables:
```bash
PROJECT_ID="$(gcloud config get-value project)"
REGION="us-central1"
FUNCTION_NAME="fetch_wsj"
SERVICE_NAME_INGESTOR="WSJ RSS ingestor"
SERVICE_NAME_SCHEDULER="WSJ RSS scheduler"
BUCKET_NAME="ita-wsj-rss-jsonl-bucket"
BASE_PREFIX="rss-landing/source=wsj"
```
Note: some commands interchange `REGION` and `LOCATION`

Count lines
```bash
gcloud storage ls -r gs://ita-wsj-rss-jsonl-bucket/rss-landing/source=wsj/ \
  | grep '\.jsonl$' \
  | while read -r obj; do
      gcloud storage cat "$obj" | wc -l
    done \
  | awk '{sum += $1} END {print sum}'
```
Note: `read -r` means read the line literally (don't treat backslashes as escape characters), 
Note2: `grep -v '/$'` to end directories, where `-v` means invert the match

Concat
```bash
gcloud storage cat gs://ita-wsj-rss-jsonl-bucket/rss-landing/source=wsj/year=YYYY/month=MM/day=DD/hour=HH/minute=MM/run-...jsonl | head -n 1
```

# CRON

Link: [Cron](https://crontab.guru)

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

List buckets/content with access:
```bash
gsutil ls
gsutil ls gs://ita-techcrunch-rss-jsonl-bucket/
gsutil ls gs://ita-techcrunch-rss-jsonl-bucket/rss-landing/source=techcrunch/`
gsutil ls -r gs://ita-techcrunch-rss-jsonl-bucket/
gsutil ls -l gs://ita-techcrunch-rss-jsonl-bucket/
gsutil ls -l -r gs://ita-techcrunch-rss-jsonl-bucket/ | grep '\.jsonl$'
gsutil ls -l -r gs://ita-techcrunch-rss-jsonl-bucket/ \
  | grep -v '/$' \
  | sort -k2,2 -k3,3 \
  | tail -n 20
gsutil ls -l -r gs://ita-techcrunch-rss-jsonl-bucket/ \
  | awk 'BEGIN{c=0;b=0} /^[[:space:]]*[0-9]+/ {c++; b+=$1} END{printf "files=%d\nbytes=%d\nGB=%.2f\n", c, b, b/1024/1024/1024}'
```
Note: `-k2,2` means `-k defines a key, switch into key-parsing mode`, `start`, `end`

### Create and View Bucket

```bash
gcloud config set project YOUR_PROJECT_ID
gcloud storage buckets create gs://ita-wsj-rss-jsonl-bucket \
  --location=us-central1 \
  --uniform-bucket-level-access
gcloud storage buckets list --filter="name:ita-wsj-rss-jsonl-bucket"
```

### Storage Utility v. gcloud

`gsutil` = Google Storage utility

List; 

```bash
gsutil rsync -r ./local/ gs://my-bucket/local/

gsutil -m rm -r gs://ita-wsj-rss-jsonl-bucket/rss-landing
gcloud storage rm --recursive gs://ita-wsj-rss-jsonl-bucket/rss-landing/

gsutil cp file.txt gs://my-bucket/
gsutil -m cp -r "gs://$BUCKET/rss-landing/source=wsj/" ./wsj_download/
gcloud storage cp --recursive "gs://$BUCKET/rss-landing/source=wsj/" ./wsj_download/
```
Note: `-m` means multi-threaded, gcloud handles this

### Service Accounts

A service account is basically a robot identity in Google Cloud. Humans log in with user accounts (your email). Cloud Functions, Cloud Run, schedulers, VMs, pipelines, etc. need a way to authenticate too—so they use service accounts.

Typically, there are two service accounts:

#### Runtime service account (Function’s identity)

Your function runs “as” this account. If it needs to write JSONL files into your bucket, you grant it something like:
`roles/storage.objectAdmin` on that bucket.

#### Caller service account (Scheduler’s identity)

Cloud Scheduler calls your HTTP function using OIDC (a signed identity token). That service account needs permission to invoke the function:
`roles/run.invoker` (for Gen2 functions, because they run on Cloud Run).

See service accounts:
```bash
gcloud iam service-accounts list
```

Establish service accounts:
Runtime function SA can write to the bucket, 
```bash
gcloud storage buckets add-iam-policy-binding gs://ita-wsj-rss-jsonl-bucket \
  --member="serviceAccount:wsj-rss-ingestor@ita-development-project.iam.gserviceaccount.com" \
  --role="roles/storage.objectAdmin"
```

Scheduler SA can invoke the Gen2 function (Cloud Run invoker), 
```bash
gcloud projects add-iam-policy-binding ita-development-project \
  --member="serviceAccount:wsj-rss-scheduler@ita-development-project.iam.gserviceaccount.com" \
  --role="roles/run.invoker"
```

Scheduler SA can invoke the Gen2 function (Cloud Run invoker)
```bash
PROJECT_ID="ita-development-project"
PROJECT_NUMBER="$(gcloud projects describe "$PROJECT_ID" --format='value(projectNumber)')"
SCHED_AGENT="service-${PROJECT_NUMBER}@gcp-sa-cloudscheduler.iam.gserviceaccount.com"

gcloud iam service-accounts add-iam-policy-binding \
  wsj-rss-scheduler@ita-development-project.iam.gserviceaccount.com \
  --member="serviceAccount:${SCHED_AGENT}" \
  --role="roles/iam.serviceAccountTokenCreator"
```

IAM Permissions ?
```bash
gsutil iam ch ...
gsutil setmeta ...
```

### Cloud Functions

Deploy the Gen2 Cloud Function:
```bash
gcloud functions deploy fetch_wsj \
  --gen2 \
  --region=us-central1 \
  --runtime=python311 \
  --entry-point=fetch_wsj \
  --source=. \
  --trigger-http \
  --no-allow-unauthenticated \
  --service-account="wsj-rss-ingestor@ita-development-project.iam.gserviceaccount.com" \
  --max-instances=1 \
  --timeout=540s \
  --set-env-vars="BUCKET_NAME=ita-wsj-rss-jsonl-bucket,BASE_PREFIX=rss-landing,DEDUP_FILES=50,DEDUP_GUIDS=10000,USER_AGENT=ita-wsj-rss-ingestor/1.0"
```

Get the function URL:
```bash
FUNCTION_URL="$(gcloud functions describe fetch_wsj --gen2 --region=us-central1 --format='value(serviceConfig.uri)')"
echo "$FUNCTION_URL"
```

Test and evoke the function manually:
```bash
curl -s -H "Authorization: Bearer $(gcloud auth print-identity-token)" "$FUNCTION_URL"
curl -i "$FUNCTION_URL"
```
Notes: `curl` is the http request (get by defult), `-s` is silent, `-H` is header, `-i` is to include the response headers

### Scheduler

Create:
```bash
gcloud scheduler jobs create http wsj-rss-every-2h \
  --location=us-central1 \
  --schedule="0 */2 * * *" \
  --time-zone="Etc/UTC" \
  --uri="$FUNCTION_URL" \
  --http-method=GET \
  --oidc-service-account-email="wsj-rss-scheduler@ita-development-project.iam.gserviceaccount.com"
```

Verify:
```bash
gcloud scheduler jobs describe wsj-rss-every-2h --location=us-central1 \
  --format="yaml(name,schedule,timeZone,httpTarget.uri,httpTarget.oidcToken.serviceAccountEmail)"
```

Trigger:
```bash
gcloud scheduler jobs run wsj-rss-every-2h --location=us-central1
```

Pause/Resume:
```bash
gcloud scheduler jobs pause wsj-rss-every-2h --location=us-central1
gcloud scheduler jobs resume wsj-rss-every-2h --location=us-central1
```

Inspect:
```bash
gcloud storage ls -l -r gs://ita-wsj-rss-jsonl-bucket/rss-landing/source=wsj/ \
  | grep '\.jsonl$' \
  | sort -k2,2 -k3,3 \
  | tail -n 20
```
See top of page.

Download:
```bash
mkdir -p wsj_download
gcloud storage cp -r \
  gs://ita-wsj-rss-jsonl-bucket/rss-landing/source=wsj/ \
  wsj_download/
```

### Logs

Examlple
```bash
gcloud functions logs read fetch_wsj --gen2 --region us-central1 --limit 200

gcloud logging read \
  'resource.type="cloud_run_revision" AND resource.labels.service_name="fetch_wsj"' \
  --limit=200 \
  --order=asc \
  --format="value(timestamp,textPayload)" \
  | tail -n 50
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

Deply a function: 
```bash
gcloud functions deploy "$FUNCTION_NAME" \
  --gen2 \
  --region="$REGION" \
  --runtime=python311 \
  --source="." \
  --entry-point="fetch_wsj" \
  --trigger-http \
  --no-allow-unauthenticated \
  --set-env-vars="BUCKET_NAME=$BUCKET_NAME,BASE_PREFIX=$BASE_PREFIX"
```

Test a function:
```bash
FUNCTION_URL="$(gcloud functions describe "$FUNCTION_NAME" --gen2 --region="$REGION" --format='value(serviceConfig.uri)')"
echo "$FUNCTION_URL"

curl -s -X POST "$FUNCTION_URL" \
  -H "Authorization: Bearer $(gcloud auth print-identity-token)" \
  -H "Content-Type: application/json" \
  -d '{}' | python -m json.tool
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
