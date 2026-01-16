# Lambda CI/CD Implementation - Questions & Answers

A comprehensive list of questions to clarify before implementing the Lambda CI/CD pipeline.

---

## 1. Prerequisites & Setup

### 1.1 What do I need before starting this implementation?
- GitHub repository with admin access
- AWS account with Lambda, IAM, and S3 permissions
- AWS CLI configured locally (for testing)
- GitHub Secrets: `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`

### 1.2 Do I need to install anything locally?
No. The CI/CD runs entirely on GitHub Actions runners. Local tools are optional for testing.

### 1.3 What AWS permissions does the deployment IAM user need?
```
lambda:CreateFunction, lambda:UpdateFunctionCode, lambda:UpdateFunctionConfiguration
lambda:PublishVersion, lambda:GetFunction, lambda:ListFunctions
iam:CreateRole, iam:AttachRolePolicy, iam:PassRole, iam:GetRole
s3:GetObject, s3:PutObject (for deployment artifacts)
logs:CreateLogGroup, logs:CreateLogStream, logs:PutLogEvents
```

### 1.4 Where do I configure the GitHub Secrets?
Repository → Settings → Secrets and variables → Actions → New repository secret

---

## 2. Directory Structure

### 2.1 Why does the README mention `lambda/` but our code is in `data_ingestion_pipeline/lambdas/`?
The README was a specification template. We'll adapt the workflow to scan your current directories.

### 2.2 Do I need to move existing Lambda folders?
**No**. The workflow can be configured to scan multiple directories:
- `data_ingestion_pipeline/lambdas/`
- `onboarding_pipeline/lambda/`

### 2.3 What happens if I add a new Lambda folder?
The workflow will automatically detect it and create the Lambda in AWS on the next deployment.

### 2.4 Can I have nested folders inside a Lambda directory?
Yes, but only the root `lambda_function.py` and `config.json` are used. Dependencies can be in subfolders.

### 2.5 Are the bundled dependencies (boto3, pydantic folders) included in the ZIP?
Yes, everything in the Lambda folder is packaged except:
- `.git*` files
- `__pycache__/`
- `*.pyc` files
- `*.zip` files (existing packages)

---

## 3. Configuration (config.json)

### 3.1 What is config.json and why do I need it?
It defines Lambda settings (runtime, memory, timeout) that the workflow uses. Without it, defaults are applied.

### 3.2 What are the required fields in config.json?
```json
{
  "handler": "lambda_function.lambda_handler",
  "runtime": "python3.11",
  "timeout": 60,
  "memory_size": 128
}
```

### 3.3 What does the `#` prefix mean in environment_variables?
Keys starting with `#` are **comments** and are ignored. Useful for documenting expected variables.

### 3.4 How do environment variables get set?
- Keys without `#` are passed to `aws lambda update-function-configuration`
- Sensitive values should be stored in AWS Secrets Manager, not in config.json

### 3.5 Can I have different configs for different environments (qa/stg/prod)?
Currently, no. One config.json per Lambda. You can:
- Use SSM Parameter Store for environment-specific values
- Add `config.qa.json`, `config.stg.json` support (requires workflow modification)

### 3.6 What runtime versions are supported?
- Python: `python3.9`, `python3.10`, `python3.11`, `python3.12`
- Node.js: `nodejs18.x`, `nodejs20.x`

### 3.7 What is the maximum timeout allowed?
15 minutes (900 seconds) for standard Lambda functions.

---

## 4. GitHub Actions Workflow

### 4.1 When does the workflow run?
| Event | Branches | Condition |
|-------|----------|-----------|
| PR Merged | integration, staging, main | Only on merge (not just close) |
| Push | lambda-CICD | Every push |
| Manual | Any | Via GitHub Actions UI |

### 4.2 What if I push directly to main without a PR?
The workflow only triggers on **PR merge** for `integration`, `staging`, `main`. Direct pushes are ignored (intentionally).

### 4.3 How does change detection work?
```bash
git diff --name-only $BEFORE_SHA $AFTER_SHA
```
Compares files changed between commits. Only modified Lambda folders are deployed.

### 4.4 What if I want to force-deploy all Lambdas?
Add a manual trigger option in the workflow with a "deploy all" checkbox (requires workflow modification).

### 4.5 How long does deployment take?
- Single Lambda: ~30-60 seconds
- Multiple Lambdas: Run in parallel, ~1-2 minutes total
- With large dependencies: Up to 3-5 minutes

### 4.6 What happens if the workflow fails mid-deployment?
- Already deployed Lambdas remain updated
- Failed Lambda stays at previous version
- Re-run the workflow to retry

### 4.7 Can I deploy to multiple AWS accounts?
Currently, no. One AWS account per workflow. For multi-account, you'd need:
- Separate workflows per account, OR
- AWS Organizations cross-account roles

---

## 5. AWS Infrastructure

### 5.1 Do the IAM roles need to exist before running the workflow?
**Option A**: Pre-create roles manually (simpler, more secure)
**Option B**: Let the workflow create them (requires `iam:CreateRole` permission)

### 5.2 What are the exact IAM role names needed?
| Environment | Role Name |
|-------------|-----------|
| QA | `mmm_Dev_lambda_exec` |
| Staging | `mmm_stg_lambda_exec` |
| Production | `mmm_prod_lambda_exec` |
| CI/CD Testing | `mmm_lambda-cicd_lambda_exec` |

### 5.3 What policies should the Lambda execution roles have?
Minimum:
```
AWSLambdaBasicExecutionRole (for CloudWatch logs)
```
For data ingestion Lambdas:
```
AmazonS3FullAccess (or scoped bucket access)
AmazonDynamoDBFullAccess (or scoped table access)
```

### 5.4 Which AWS region is used?
`eu-west-1` (Ireland). Hardcoded in the workflow. Change `--region` flag if needed.

### 5.5 What S3 buckets are required?
None for the CI/CD itself. The Lambdas use:
- `mmm-{env}-audit-logs`
- `mmm-{env}-data-stage`
- `prd-mm-vendor-sync` (Tracer bucket)

### 5.6 What DynamoDB tables are required?
- `mmm-{env}-pipeline-infos`
- `mmm-{env}-client-metadata`
- `mmm-{env}-data-transfer-logs`

---

## 6. Deployment Process

### 6.1 How are Lambda names generated?
`{folder-name}-{environment-suffix}`

| Folder | Branch | Lambda Name |
|--------|--------|-------------|
| data_ingestion_slack | integration | `data_ingestion_slack-qa` |
| mmm_dev_data_transfer | main | `mmm_dev_data_transfer-prod` |

### 6.2 What if the Lambda name already exists in AWS?
The workflow **updates** it. No duplicate is created.

### 6.3 Can I deploy to a specific Lambda without affecting others?
Yes, only Lambdas with file changes are deployed. Others are skipped.

### 6.4 How do I rollback a bad deployment?
**Option 1**: Revert the PR and merge (triggers new deployment)
**Option 2**: Use AWS Console to deploy previous version
**Option 3**: Use `/rollback` workflow (if implemented)

### 6.5 What metadata is attached to each version?
```json
{
  "deployed_at": "2026-01-15T12:00:00Z",
  "branch": "integration",
  "commit_sha": "abc123def456"
}
```

### 6.6 Are Lambda aliases used?
Not in the current design. Versions are published, but aliases (like `$LATEST`, `PROD`) aren't set.

### 6.7 What's the package size limit?
- Direct upload: 50 MB (zipped)
- Via S3: 250 MB (unzipped)

---

## 7. Security & Permissions

### 7.1 Are AWS credentials stored in the repository?
**No**. They're stored as GitHub Secrets, encrypted at rest.

### 7.2 Who can trigger the deployment?
Anyone who can:
- Merge PRs to protected branches
- Push to `lambda-CICD` branch
- Run workflows manually (if allowed)

### 7.3 Can I restrict which Lambdas can be deployed?
Yes, by:
- Adding folder-level CODEOWNERS
- Using branch protection rules
- Adding approval gates in the workflow

### 7.4 Is there an approval step before production deployment?
Not in the current design. To add one:
- Use GitHub Environments with required reviewers
- Add a manual approval job before prod deployment

### 7.5 How do I audit who deployed what?
- GitHub Actions logs show user, timestamp, changes
- Lambda versions have commit SHA metadata
- CloudTrail logs all AWS API calls

---

## 8. Testing & Troubleshooting

### 8.1 How do I test the workflow without deploying to AWS?
- Push to `lambda-CICD` branch (test environment)
- Use `dry-run` mode (if implemented)
- Comment out AWS commands, just run detection

### 8.2 What if "Lambda function not found" error?
- First deployment: Expected, workflow will create it
- Subsequent deployment: Check function name matches

### 8.3 What if "Access Denied" error?
Check:
1. GitHub Secrets are correctly set
2. IAM user has required permissions
3. IAM role exists for the environment

### 8.4 What if "Package too large" error?
- Exclude unnecessary files (tests, docs, .git)
- Use Lambda Layers for common dependencies
- Consider S3-based deployment

### 8.5 How do I see deployment logs?
GitHub → Actions → Select workflow run → Click on job → Expand steps

### 8.6 Can I test a Lambda locally before deploying?
Yes, use:
```bash
# AWS SAM
sam local invoke -e event.json

# Docker
docker run -v $(pwd):/var/task lambci/lambda:python3.11 lambda_function.lambda_handler
```

### 8.7 What if the workflow hangs or times out?
- Default timeout: 6 hours (GitHub Actions limit)
- Check `cli-read-timeout` for AWS CLI commands
- Retry logic handles transient AWS failures

---

## 9. Questions For Your Team / Stakeholders

### Before Implementation
1. ✅ Are the IAM role names correct? (`mmm_Dev_lambda_exec`, etc.)
2. ✅ Are GitHub Secrets configured?
3. ✅ Who should have merge permissions to each branch?
4. ✅ Do we need approval gates for production?

### During Implementation
5. Should we keep current directory structure or consolidate to `lambda/`?
6. Do we need environment-specific config files?
7. Should we add Lambda Layers for shared dependencies?
8. Do we want Slack/email notifications on deployment?

### After Implementation
9. Who monitors failed deployments?
10. What's the rollback procedure?
11. How often should we audit Lambda versions?
12. Should we implement canary deployments?

---

## Quick Reference Checklist

```
□ GitHub Secrets configured (AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY)
□ IAM roles exist for each environment
□ Lambda execution roles have required policies
□ Branch protection rules set for main/staging
□ config.json created for each Lambda
□ Test workflow on lambda-CICD branch first
□ Document rollback procedure
□ Set up monitoring/alerting
```
