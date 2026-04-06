# DevOps Project 1 — GitHub Actions CI/CD Pipeline → AWS S3

A production-style CI/CD pipeline that automatically lints, builds, and deploys a static site to AWS S3 on every push to `main`. Built to demonstrate real DevOps practices: job dependencies, artifact passing, branch protection, IAM least-privilege, and environment-gated deployments.

---

## Pipeline Architecture

```
Push to main
     │
     ▼
┌─────────────┐
│  lint job   │  Validates HTML structure + required files
└──────┬──────┘
       │ needs: lint
       ▼
┌─────────────┐
│  build job  │  Copies src → dist, injects build metadata (timestamp + commit SHA)
└──────┬──────┘  Uploads dist/ as a GitHub Actions artifact
       │ needs: build  (only on push to main)
       ▼
┌─────────────┐
│ deploy job  │  Downloads artifact, configures AWS credentials, syncs to S3
└─────────────┘  Verifies deployment with aws s3 ls
```

**Key behaviors:**
- Pull requests → runs lint + build only (no deploy)
- Push to main → runs full pipeline including deploy
- Jobs are chained with `needs:` — a failed lint blocks the build, a failed build blocks deploy
- Build artifact is passed between jobs (not re-built in deploy)

---

## Tech Stack

| Layer | Tool |
|---|---|
| CI/CD | GitHub Actions |
| Hosting | AWS S3 static website |
| Auth | AWS IAM (least-privilege deployment user) |
| Secrets | GitHub Encrypted Secrets |
| Artifact storage | GitHub Actions artifacts (7-day retention) |

---

## AWS Setup (one-time)

### 1. Create an S3 bucket

```bash
aws s3 mb s3://YOUR-BUCKET-NAME --region us-east-1

# Enable static website hosting
aws s3 website s3://YOUR-BUCKET-NAME \
  --index-document index.html \
  --error-document index.html

# Set bucket policy for public read
aws s3api put-bucket-policy \
  --bucket YOUR-BUCKET-NAME \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::YOUR-BUCKET-NAME/*"
    }]
  }'
```

### 2. Create an IAM user for deployments (least privilege)

```bash
# Create the user
aws iam create-user --user-name github-actions-deployer

# Create a policy (save as policy.json first)
cat > policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::YOUR-BUCKET-NAME",
        "arn:aws:s3:::YOUR-BUCKET-NAME/*"
      ]
    }
  ]
}
EOF

aws iam put-user-policy \
  --user-name github-actions-deployer \
  --policy-name s3-deploy-policy \
  --policy-document file://policy.json

# Create access keys (save these — shown once only)
aws iam create-access-key --user-name github-actions-deployer
```

### 3. Add secrets to GitHub

Go to your repo → Settings → Secrets and variables → Actions → New repository secret

| Secret name | Value |
|---|---|
| `AWS_ACCESS_KEY_ID` | Access key from step 2 |
| `AWS_SECRET_ACCESS_KEY` | Secret key from step 2 |
| `S3_BUCKET_NAME` | Your bucket name |

---

## Branch Protection Setup

Go to repo → Settings → Branches → Add rule → Branch name: `main`

Check these boxes:
- [x] Require a pull request before merging
- [x] Require status checks to pass before merging
  - Add: `Lint & Validate`
  - Add: `Build`
- [x] Require branches to be up to date before merging

This ensures no code reaches `main` without passing the pipeline.

---

## Local Development

```bash
# Clone the repo
git clone https://github.com/YOUR_USERNAME/devops-project1.git
cd devops-project1

# Open locally
open src/index.html

# Make a change and push
git checkout -b feature/my-change
# ... make changes ...
git add .
git commit -m "feat: update content"
git push origin feature/my-change
# Open a PR — pipeline runs automatically
```

---

## Project Structure

```
devops-project1/
├── .github/
│   └── workflows/
│       └── deploy.yml        # Full CI/CD pipeline definition
├── src/
│   ├── index.html            # Static site with build metadata placeholders
│   └── style.css             # Styling
└── README.md                 # This file
```

---

## What This Demonstrates

- **CI/CD pipeline design** — 3-job chain with proper dependencies and gates
- **AWS S3 static hosting** — bucket creation, website config, public policy
- **IAM least privilege** — deployment user with minimum required permissions only
- **GitHub Actions best practices** — secrets management, artifact passing, environment URLs
- **Branch protection** — status checks required before merge to main
- **Build metadata injection** — timestamp and commit SHA baked into deployed artifact
- **Pull request workflow** — lint + build on PRs, full deploy only on main

---

*Part of a DevOps portfolio — see also [Project 2: Terraform AWS Infrastructure](../devops-project2) and [Project 3: Docker + ECS Deployment](../devops-project3)*
