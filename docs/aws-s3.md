# aws s3 — day-to-day

Practical command reference for working with S3 from the AWS CLI. The high-level `aws s3` commands cover most day-to-day operations (list, copy, sync, move, delete, presign); a handful of bucket- and object-level settings (metadata headers, ACLs, policies, encryption, versioning, lifecycle) are only exposed via the lower-level `aws s3api`.

Every command assumes `--profile default --region us-east-1` — adjust to your environment.

## 1. List buckets and objects

List every bucket the credentials can see:

```bash
aws s3 ls --profile default --region us-east-1
```

List the top-level contents of a single bucket under a prefix:

```bash
aws s3 ls s3://my-bucket/uploads/ --profile default --region us-east-1
```

Walk the entire prefix recursively, with sizes in KB/MB/GB and a final totals line:

```bash
aws s3 ls s3://my-bucket/uploads/ --recursive --human-readable --summarize --profile default --region us-east-1
```

## 2. Make and delete buckets

Create a new bucket in a specific region. The region is part of the bucket's identity — picking the wrong one is hard to undo:

```bash
aws s3 mb s3://my-bucket --region us-east-1 --profile default
```

Delete a bucket. `--force` empties non-versioned objects first; for versioned buckets you also have to delete the versions via `aws s3api delete-object --version-id`:

```bash
aws s3 rb s3://my-bucket --force --profile default --region us-east-1
```

## 3. Copy single files

Upload a local file to a specific key:

```bash
aws s3 cp ./report.pdf s3://my-bucket/uploads/report.pdf --profile default --region us-east-1
```

Download an object to the local filesystem:

```bash
aws s3 cp s3://my-bucket/uploads/report.pdf ./report.pdf --profile default --region us-east-1
```

Recursively copy a whole directory tree, picking up only PDFs and skipping draft files:

```bash
aws s3 cp ./local-dir s3://my-bucket/uploads/ --recursive --exclude "*" --include "*.pdf" --exclude "*draft*" --profile default --region us-east-1
```

## 4. Sync directories

Mirror a local directory into a prefix. `--delete` removes destination objects that no longer exist on the source — leave it off if you only ever want to add/update:

```bash
aws s3 sync ./local-dir s3://my-bucket/uploads/ --delete --exclude "*.tmp" --include "*" --profile default --region us-east-1
```

The reverse direction works the same way — handy for pulling a snapshot of a prefix into a fresh working copy:

```bash
aws s3 sync s3://my-bucket/uploads/ ./local-dir --profile default --region us-east-1
```

## 5. Move and delete

Move (copy + delete source) — rename keys or migrate between buckets:

```bash
aws s3 mv s3://my-bucket/uploads/report.pdf s3://my-bucket/archive/report.pdf --profile default --region us-east-1
```

Delete a single object:

```bash
aws s3 rm s3://my-bucket/uploads/report.pdf --profile default --region us-east-1
```

Delete every object under a prefix. Try it with `--dryrun` first:

```bash
aws s3 rm s3://my-bucket/uploads/ --recursive --dryrun --profile default --region us-east-1
```

## 6. Presigned URLs

Generate a time-limited GET URL anyone can use without AWS credentials. Default expiry is 3600s; max is 604800s (7 days):

```bash
aws s3 presign s3://my-bucket/uploads/report.pdf --expires-in 3600 --profile default --region us-east-1
```

## 7. Object metadata and ACL (s3api)

Inspect an object's metadata without downloading it — size, content-type, ETag, server-side-encryption, custom `x-amz-meta-*` headers:

```bash
aws s3api head-object --bucket my-bucket --key uploads/report.pdf --profile default --region us-east-1
```

Upload an object with custom metadata and an explicit content-type. Note `--metadata` keys become `x-amz-meta-<key>` request headers:

```bash
aws s3api put-object \
  --bucket my-bucket \
  --key uploads/report.pdf \
  --body ./report.pdf \
  --content-type application/pdf \
  --metadata source=cheatsheet,owner=team-platform \
  --profile default --region us-east-1
```

Make an existing object publicly readable (only works on buckets without "bucket-owner-enforced" ACL settings):

```bash
aws s3api put-object-acl --bucket my-bucket --key uploads/report.pdf --acl public-read --profile default --region us-east-1
```

## 8. Bucket policy and default encryption

Print the current bucket policy as JSON:

```bash
aws s3api get-bucket-policy --bucket my-bucket --profile default --region us-east-1 --output text --query Policy
```

Set default server-side encryption to SSE-S3 (AES256). For SSE-KMS pass `SSEAlgorithm=aws:kms,KMSMasterKeyID=<arn>`:

```bash
aws s3api put-bucket-encryption \
  --bucket my-bucket \
  --server-side-encryption-configuration '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}' \
  --profile default --region us-east-1
```

## 9. Versioning and lifecycle

Enable versioning — every PUT thereafter writes a new version instead of overwriting. Allow ~15 minutes for the setting to propagate before relying on it:

```bash
aws s3api put-bucket-versioning --bucket my-bucket --versioning-configuration Status=Enabled --profile default --region us-east-1
```

Read the current lifecycle rules (transition / expiration policies):

```bash
aws s3api get-bucket-lifecycle-configuration --bucket my-bucket --profile default --region us-east-1
```

## 10. Pipe to stdin/stdout

`-` as the source or destination streams the body through stdin/stdout — useful for piping or for objects you never want on disk.

Upload from stdin (here, a local file via shell redirection):

```bash
aws s3 cp - s3://my-bucket/uploads/report.pdf < ./report.pdf --profile default --region us-east-1
```

Download to stdout (and on to disk, jq, gzip, etc.):

```bash
aws s3 cp s3://my-bucket/uploads/report.pdf - > ./report.pdf --profile default --region us-east-1
```
