# Lambda — Deployment Guide

How to build, package, and deploy the image Lambda to AWS.

---

## Prerequisites

- AWS CLI configured with `tensor` profile (region `eu-north-1`)
- Node.js 20+
- The function `tensor-labz-image-handler` already exists in AWS

---

## Manual deploy (standard workflow)

```bash
cd tensor-labz-image-lambda

# 1. Install dependencies
npm install

# 2. Build + zip
npm run build:zip
# Creates: function.zip (contains dist/index.js)

# 3. Upload to Lambda
aws --profile tensor lambda update-function-code \
  --region eu-north-1 \
  --function-name tensor-labz-image-handler \
  --zip-file fileb://function.zip

# 4. Wait for update to complete
aws --profile tensor lambda wait function-updated \
  --region eu-north-1 \
  --function-name tensor-labz-image-handler

echo "Deploy complete ✓"
```

---

## Automatic deploy (GitHub Actions)

The repo has a CI/CD workflow at `.github/workflows/deploy.yml`. It triggers on every push to `master`:

1. Checks out the code
2. Installs dependencies
3. Runs `npm run build:zip`
4. Calls `aws lambda update-function-code`

Secrets required in the GitHub repo settings:

| Secret                  | Value               |
| ----------------------- | ------------------- |
| `AWS_ACCESS_KEY_ID`     | IAM user access key |
| `AWS_SECRET_ACCESS_KEY` | IAM user secret     |

The IAM user needs `lambda:UpdateFunctionCode` permission on the `tensor-labz-image-handler` ARN.

---

## Setting / updating environment variables

```bash
aws --profile tensor lambda update-function-configuration \
  --region eu-north-1 \
  --function-name tensor-labz-image-handler \
  --environment "Variables={
    SUPABASE_JWT_SECRET=<from supabase settings>,
    S3_BUCKET=tensor-labz-store,
    S3_REGION=eu-north-1,
    CDN_URL=https://tensor-labz-store.s3.eu-north-1.amazonaws.com,
    ALLOWED_ORIGIN=https://tensorlabz.com
  }"
```

!!! warning "Updating env vars restarts the Lambda"
There is a brief cold-start delay after updating environment variables.

---

## First-time Lambda creation

If you ever need to recreate the function from scratch:

```bash
# Create execution role (one-time)
aws --profile tensor iam create-role \
  --role-name tensor-labz-lambda-exec \
  --assume-role-policy-document '{
    "Version":"2012-10-17",
    "Statement":[{
      "Effect":"Allow",
      "Principal":{"Service":"lambda.amazonaws.com"},
      "Action":"sts:AssumeRole"
    }]
  }'

# Attach basic execution policy
aws --profile tensor iam attach-role-policy \
  --role-name tensor-labz-lambda-exec \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

# Attach S3 permissions
aws --profile tensor iam put-role-policy \
  --role-name tensor-labz-lambda-exec \
  --policy-name s3-access \
  --policy-document '{
    "Version":"2012-10-17",
    "Statement":[{
      "Effect":"Allow",
      "Action":["s3:PutObject","s3:DeleteObject"],
      "Resource":"arn:aws:s3:::tensor-labz-store/*"
    }]
  }'

# Build first
cd tensor-labz-image-lambda && npm run build:zip

# Create function
aws --profile tensor lambda create-function \
  --region eu-north-1 \
  --function-name tensor-labz-image-handler \
  --runtime nodejs20.x \
  --handler index.handler \
  --role arn:aws:iam::<ACCOUNT_ID>:role/tensor-labz-lambda-exec \
  --zip-file fileb://function.zip \
  --memory-size 256 \
  --timeout 15
```

---

## Verifying a deployment

```bash
# Invoke a health-check OPTIONS preflight
curl -X OPTIONS \
  https://ewf03ybvmc.execute-api.eu-north-1.amazonaws.com/image/upload-url \
  -v 2>&1 | grep -E "HTTP|Access-Control"
# Expect: HTTP/2 204 and Access-Control-Allow-* headers

# Check function config
aws --profile tensor lambda get-function-configuration \
  --region eu-north-1 \
  --function-name tensor-labz-image-handler \
  --query '[LastModified, Runtime, MemorySize, Timeout, State]'
```
