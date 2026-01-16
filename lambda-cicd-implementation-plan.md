# Lambda CI/CD Pipeline Implementation Plan

Implement automated GitHub Actions workflow for deploying Lambda functions with multi-environment support, change detection, and automatic versioning for the MikGorilla.AI Marketing Mix Modeling platform.

---

## User Review Required

> [!IMPORTANT]
> **Directory Structure Decision**: The current Lambdas are in `data_ingestion_pipeline/lambdas/` and `onboarding_pipeline/lambda/`. The README specifies `lambda/` as the target directory. Should we:
> 1. **Option A**: Move all Lambdas to a new `lambda/` folder (breaking change)
> 2. **Option B**: Keep current structure and modify the workflow to scan multiple directories
> 3. **Option C**: Create symlinks or reference existing locations

> [!WARNING]
> **IAM Role Names**: The README specifies role names like `mmm_Dev_lambda_exec`, `mmm_stg_lambda_exec`, `mmm_prod_lambda_exec`. Please confirm these exact IAM role names exist in your AWS account, or provide the correct role names.

> [!CAUTION]
> **AWS Credentials**: The workflow requires `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` as GitHub Secrets. Ensure these are configured in your GitHub repository settings before deploying.

---

## Current State Analysis

### Existing Lambda Functions

| Lambda | Location | Purpose | Has config.json |
|--------|----------|---------|-----------------|
| `data_ingestion_slack` | `data_ingestion_pipeline/lambdas/` | Slack notifications for pipeline status | ❌ |
| `mmm_dev_data_transfer` | `data_ingestion_pipeline/lambdas/` | S3 data transfer with pandas processing | ❌ |
| `mmm_dev_get_client` | `data_ingestion_pipeline/lambdas/` | Client data retrieval | ❌ |
| `stale_data_check` | `data_ingestion_pipeline/lambdas/` | Stale data detection | ❌ |
| `onboarding` | `onboarding_pipeline/lambda/` | Onboarding data handler | ❌ |

### Missing Components
- `.github/workflows/deploy-lambda.yaml` - No CI/CD workflow exists
- `config.json` files - None of the Lambdas have configuration files

---

## Proposed Changes

### Component 1: GitHub Workflows Setup

#### [NEW] deploy-lambda.yaml

Create the main CI/CD workflow with:
- **Triggers**: PR merge to `integration`, `staging`, `main` branches + push to `lambda-CICD`
- **Jobs**:
  1. `detect-changes` - Identify new/modified Lambda functions
  2. `validate` - Check Lambda existence in AWS
  3. `deploy` - Create or update Lambda functions
  4. `publish-version` - Version lambdas with metadata

```yaml
# Key workflow structure
name: Deploy Lambda Functions
on:
  pull_request:
    types: [closed]
    branches: [integration, staging, main]
  push:
    branches: [lambda-CICD]

jobs:
  detect-changes:
    # Identify which lambdas have changes
  validate:
    # Check AWS for existing functions
  deploy:
    # Package and deploy lambdas
  publish-version:
    # Create versions with metadata
```

---

### Component 2: Lambda Configuration Files

Each Lambda requires a `config.json` with runtime settings.

#### data_ingestion_slack/config.json
```json
{
  "handler": "lambda_function.lambda_handler",
  "timeout": 30,
  "runtime": "python3.11",
  "memory_size": 256,
  "environment_variables": {
    "SLACK_WEBHOOK_URL": "",
    "LOG_LEVEL": "INFO"
  }
}
```

#### mmm_dev_data_transfer/config.json
```json
{
  "handler": "lambda_function.lambda_handler",
  "timeout": 900,
  "runtime": "python3.11",
  "memory_size": 1024,
  "environment_variables": {
    "#ENVIRONMENT": "dev",
    "#PIPELINE_INFO_TABLE": "mmm-dev-pipeline-infos",
    "#TRANSFER_LOGS_TABLE": "mmm-dev-data-transfer-logs",
    "#AUDIT_LOGS_BUCKET": "mmm-dev-audit-logs",
    "LOG_LEVEL": "INFO"
  }
}
```

#### mmm_dev_get_client/config.json
```json
{
  "handler": "lambda_function.lambda_handler",
  "timeout": 60,
  "runtime": "python3.11",
  "memory_size": 256,
  "environment_variables": {
    "LOG_LEVEL": "INFO"
  }
}
```

#### stale_data_check/config.json
```json
{
  "handler": "lambda_function.lambda_handler",
  "timeout": 60,
  "runtime": "python3.11",
  "memory_size": 256,
  "environment_variables": {
    "LOG_LEVEL": "INFO"
  }
}
```

#### onboarding/config.json
```json
{
  "handler": "lambda_function.lambda_handler",
  "timeout": 30,
  "runtime": "python3.12",
  "memory_size": 512,
  "environment_variables": {
    "#CLIENT_METADATA_TABLE": "mmm-dev-client-metadata",
    "#PIPELINE_INFOS_TABLE": "mmm-dev-pipeline-infos",
    "LOG_LEVEL": "INFO"
  }
}
```

---

### Component 3: Workflow Configuration

#### Branch-to-Environment Mapping

| Branch | Suffix | Lambda Name Example | IAM Role |
|--------|--------|---------------------|----------|
| `integration` | `qa` | `data_ingestion_slack-qa` | `mmm_Dev_lambda_exec` |
| `staging` | `stg` | `data_ingestion_slack-stg` | `mmm_stg_lambda_exec` |
| `main` | `prod` | `data_ingestion_slack-prod` | `mmm_prod_lambda_exec` |
| `lambda-CICD` | `lambda-cicd` | `data_ingestion_slack-lambda-cicd` | `mmm_lambda-cicd_lambda_exec` |

---

## Workflow Implementation Details

### Change Detection Logic
```bash
# Detect changed Lambda folders
git diff --name-only ${{ github.event.before }} ${{ github.sha }} | \
  grep -E '^(data_ingestion_pipeline/lambdas|onboarding_pipeline/lambda)/' | \
  cut -d'/' -f3 | sort -u
```

### Lambda Packaging
```bash
# For Python Lambdas with requirements.txt
pip install -r requirements.txt -t .
zip -r lambda.zip . -x "*.git*" -x "__pycache__/*" -x "*.pyc"
```

### Deployment with Retry
```bash
# AWS CLI with retry logic
aws lambda update-function-code \
  --function-name $FUNCTION_NAME \
  --zip-file fileb://lambda.zip \
  --region eu-west-1 \
  --cli-read-timeout 300
```

---

## Verification Plan

### Automated Tests

1. **Workflow Syntax Validation**
   ```powershell
   # Install actionlint and validate workflow
   npm install -g @action-validator/cli
   action-validator .github/workflows/deploy-lambda.yaml
   ```

2. **Config.json Schema Validation**
   ```powershell
   # Create a test script to validate all config.json files
   python -c "
   import json
   import glob
   required_keys = ['handler', 'timeout', 'runtime', 'memory_size']
   for f in glob.glob('**/config.json', recursive=True):
       with open(f) as fp:
           config = json.load(fp)
           for key in required_keys:
               assert key in config, f'Missing {key} in {f}'
   print('All config.json files valid')
   "
   ```

### Manual Verification

1. **GitHub Actions Dry Run**
   - Push changes to `lambda-CICD` branch
   - Monitor workflow execution in GitHub Actions UI
   - Verify Lambda detection output in workflow logs

2. **AWS Deployment Verification**
   - Check AWS Lambda console for deployed functions
   - Verify environment suffix naming (e.g., `data_ingestion_slack-qa`)
   - Confirm version metadata includes commit SHA and branch

3. **User Testing** (Requires user assistance)
   - Create a test Lambda folder
   - Push to trigger deployment
   - Verify Lambda appears in AWS with correct configuration
