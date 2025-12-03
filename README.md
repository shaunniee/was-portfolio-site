# Static Portfolio Site – AWS (S3 + CloudFront + CI/CD)

**WEBSTIE LINK**: www.shaunvividszportfolio.com

Static portfolio website hosted on AWS with a production-style setup:

- Private **S3** bucket (no public access)
- **CloudFront** for global delivery + HTTPS
- **Route 53 + ACM** for custom domain and TLS
- **CodePipeline + CodeBuild** for automated deployments from Git
- **CloudFront logs + CloudWatch alarms** for basic observability

---

## 🔎 Overview

**Goal**

Serve a static portfolio site on a custom domain with:

- HTTPS by default  
- Private origin (no public S3 website hosting)  
- Git-based CI/CD (push to `main` → deploy)  
- Basic monitoring and logging so issues are visible

**Main stack**

- S3 (private content bucket)  
- CloudFront (CDN, HTTPS, caching)  
- Route 53 (DNS)  
- ACM (public cert in `us-east-1`)  
- CodePipeline + CodeBuild (CI/CD)  
- CloudWatch (metrics, alarms, logs)  
- SNS (email notifications for alarms)

---

## 🏗 Architecture

<img width="5138" height="1875" alt="Portfolio static website (2)" src="https://github.com/user-attachments/assets/5a1c27b6-06bb-4148-af41-ba14100c5baf" />


## 📁 Key AWS resources

### S3

- **Content bucket**  
  - Example: `shaun-portfolio-content`  
  - Region: `eu-west-1`  
  - Purpose: store `index.html`, CSS, JS, images  
  - Settings:
    - Block public access: **ON**  
    - Bucket policy: only CloudFront **Origin Access Control (OAC)** can `s3:GetObject`  
    - Default encryption enabled

- **CloudFront logs bucket**  
  - Example: `shaun-portfolio-cf-logs`  
  - Region: `eu-west-1`  
  - Purpose: store CloudFront access logs  
  - Settings:
    - Block public access: **ON**  

### CloudFront

- Origin: S3 content bucket (via OAC)
- Default root object: `index.html`
- Custom domain: `www.shaunvividszportfolio.com`
- TLS certificate: ACM public cert in `us-east-1`
- Viewer protocol: Redirect HTTP → HTTPS
- Logging: enabled to `shaun-portfolio-cf-logs`

### Route 53

- Hosted zone: `shaunvivivdszportfolio.com`
- Records:
  - `A` (Alias) for `www` → CloudFront distribution

### CI/CD

- **CodePipeline**: `shaunvividsz-portfolio-pipeline`
  - Source stage: GitHub(`main` branch)
  - Build stage: CodeBuild project `yourdomain-site-build`

- **CodeBuild**: `shaunvividsz-portfolio-build`
  - Source: artifact from CodePipeline
  - Buildspec: `buildspec.yml` in repo
  - IAM role permissions:
    - `s3:ListBucket`, `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject` on the content bucket
    - `cloudfront:CreateInvalidation` on the distribution
    - CloudWatch Logs access

---

## 🧩 buildspec.yml

Used by CodeBuild to deploy the site:

```yaml
version: 0.2

env:
  variables:
    S3_BUCKET: "shaun-portfolio-content"
    CLOUDFRONT_DISTRIBUTION_ID: "E2AMUPAF5FZ1M8"

phases:
  install:
    commands:
      - echo "Static site deploy"
  build:
    commands:
      - echo "No build step for plain HTML"
  post_build:
    commands:
      - echo "Syncing to S3..."
      - aws s3 sync . s3://$S3_BUCKET --delete --exclude ".git/*" --exclude "buildspec.yml"
      - echo "Invalidating CloudFront cache..."
      - aws cloudfront create-invalidation --distribution-id $CLOUDFRONT_DISTRIBUTION_ID --paths "/*"

artifacts:
  files:
    - '**/*'
```

---

## 📊 Logging & monitoring

### CloudFront access logs

- Enabled at CloudFront distribution level
- Destination: `shaun-portfolio-cf-logs` S3 bucket
- Use cases:
  - Analyse traffic patterns
  - Debug 4xx/5xx errors
  - Feed into Athena/Glue later if needed

### CloudWatch alarm – CloudFront 5xx error rate

Configured in **`us-east-1`** (CloudFront metrics region):

- Metric: `5xxErrorRate` for this distribution  
- Period: 5 minutes  
- Condition: alarm if `5xxErrorRate > 1%` for one or more periods  
- SNS topic: e.g. `svd-portfolio-alert` with email subscription  

**Purpose:**  
If the site starts returning a lot of 5xx responses (origin errors, misconfig, etc.), an email is sent so it can be investigated quickly.

### Other observability pieces

- **CodeBuild logs**:
  - CloudWatch log group: `/aws/codebuild/yourdomain-site-build` (region: `eu-west-1`)
  - Shows full output of `aws s3 sync` and `cloudfront create-invalidation`
-  Cost monitoring:
  - AWS Budgets monthly cost budget (e.g. €5–10) with email alerts on actual and forecasted spend
  - CloudWatch billing alarm on total `EstimatedCharges` (in `us-east-1`)

---

## 🚀 How to deploy

1. Make changes locally (e.g. edit `index.html`).
2. Commit and push to `main`:

   ```bash
   git add .
   git commit -m "Update portfolio"
   git push origin main
   ```

3. CodePipeline run:
   - **Source**: pulls latest code from GitHub
   - **Build**: CodeBuild runs `buildspec.yml`
4. CodeBuild:
   - Syncs files to the S3 content bucket
   - Creates a CloudFront invalidation for `/*`
5. Result: new version is served at  
   `https://www.shaunvividszporfolio.com`

---

## 🔐 Security & cost

### Security

- S3 bucket is **never public** – access only via CloudFront OAC
- HTTPS enforced end-to-end using ACM TLS
- IAM:
  - CodeBuild role has **least-privilege** permissions to:
    - Read/write to the specific content bucket
    - Create CloudFront invalidations
- Logs:
  - CloudFront access logs stored in a private S3 bucket
  - Build logs in CloudWatch

### Cost

This architecture is designed to stay cheap for a personal site:

- S3: data storage + GET requests
- CloudFront: low-traffic CDN usage
- Route 53: one hosted zone + light DNS queries
- ACM: public certificates are free
- CodePipeline/CodeBuild: pay-per-use for infrequent builds

No EC2, RDS, or continuously running compute.

---

## 📂 Repo structure

```text
.
├─ index.html              # Main static site
├─ buildspec.yml           # CI/CD deploy logic
└─ README.md               # This documentation
```
