## CAIP_02 – Image Analysis Pipeline (AWS Rekognition + GitHub Actions)
This repo provides an automated image-analysis pipeline that:
- **Uploads images from the `images/` folder to S3 via GitHub Actions**
- **Invokes Lambda functions that call Amazon Rekognition**
- **Stores analysis results in DynamoDB for later querying/auditing**

Main pieces:
- **CloudFormation** – `cloudformation/template.yml` (S3, DynamoDB, Lambda, IAM, Rekognition access)
- **Lambdas** – `lambda/prod.py`, `lambda/beta.py`
- **GitHub Actions** – `.github/workflows/` (PR + merge automation)

---

## 1. Set up AWS resources (S3, Rekognition, DynamoDB)

- **Prereqs**
  - AWS account with permissions for S3, Lambda, DynamoDB, IAM, Rekognition
  - AWS CLI installed and configured (`aws configure`)

- **Deploy CloudFormation**

```bash
aws cloudformation deploy \
  --template-file cloudformation/template.yml \
  --stack-name caip-02-image-analysis \
  --capabilities CAPABILITY_NAMED_IAM
```

This creates:
- S3 bucket for image input (used by the GitHub Action)
- IAM permissions so Lambda can call Rekognition
- DynamoDB tables (e.g., beta/prod) for analysis results
- Lambda functions wired to the S3 bucket / triggers defined in the template

After deployment, record:
- **S3 bucket name** (for `S3_ANALYZE_BUCKET`)
- **DynamoDB table names** (to verify logs)

---

## 2. Configure GitHub secrets
In the GitHub repo:
- Go to **Settings → Secrets and variables → Actions → New repository secret**
- Add:
  - **`AWS_ACCESS_KEY_ID`** – IAM access key (can write to S3 and, if needed, DynamoDB)
  - **`AWS_SECRET_ACCESS_KEY`** – Matching secret
  - **`AWS_REGION`** – Region of the stack (e.g. `us-east-1`)
  - **`S3_ANALYZE_BUCKET`** – Input S3 bucket name from the stack

The `on_pull_request.yml` workflow checks these secrets before running.

---

## 3. Add and analyze images

- **Add images**
  - Put supported files in `images/` (`*.jpg`, `*.png`, `*.gif`, `*.webp`, etc.)
  - Commit and push.

- **Trigger analysis**
  - **Pull Request to `main`**:
    - Runs **Analyze Images on Pull Request**:
      - Validates secrets
      - Uploads `images/` to `s3://$S3_ANALYZE_BUCKET/rekognition-input/beta`
      - Triggers the **beta Lambda + Rekognition** flow.
  - **Merge to `main`**:
    - `on_merge.yml` typically:
      - Uploads to a prod prefix/bucket
      - Triggers the **prod Lambda + Rekognition** flow.

Repeat any time by updating `images/` and pushing.

---

## 4. Verify DynamoDB logging

After Lambda + Rekognition run:
- **Find table names** in `cloudformation/template.yml` (beta/prod).

- **Check in AWS console**
  - **DynamoDB → Tables → [Your Table] → Explore table items**
  - Look for recent items whose attributes match:
    - Image key / S3 object path
    - Rekognition labels/metadata (labels, confidence scores, timestamps)

- **Or via AWS CLI**

```bash
aws dynamodb scan \
  --table-name YOUR_DYNAMODB_TABLE_NAME \
  --max-items 10
```

If you see items for your uploaded images, **logging to the correct DynamoDB table is working**.
