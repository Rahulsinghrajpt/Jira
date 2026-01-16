# Lambda CI/CD Deployment Guide

# Lambda CI/CD Deployment Guide (Temporary)

This document explains how to deploy AWS Lambda functions using the GitHub Actions workflow located at `.github/workflows/deploy-lambda.yaml`.

## Table of Contents

1. [Overview](#overview)
2. [Workflow Triggers](#workflow-triggers)
3. [Environment Mapping](#environment-mapping)
4. [Lambda Folder Structure](#lambda-folder-structure)
5. [Configuration Files](#configuration-files)
6. [Deployment Process](#deployment-process)
7. [Examples](#examples)
8. [Best Practices](#best-practices)
9. [Troubleshooting](#troubleshooting)
10. [Additional Resources](#additional-resources)

---

## Overview

The Lambda CI/CD workflow automatically deploys Lambda functions to AWS when changes are made to the `lambda/` directory. The workflow supports:

- **Multiple runtimes**: Python 3.11 and Node.js 20.x
- **Multiple environments**: QA, Staging, and Production
- **Automatic change detection**: Only deploys Lambdas that have been modified
- **IAM role management**: Automatically creates and configures IAM roles
- **Version publishing**: Automatically publishes versions after deployment

---

## Workflow Triggers

The workflow is triggered in the following scenarios:

### 1. Feature branch

- **Branch**: Cutoff a feature branch from `main` branch
- **Changes**: Do the required changes/setup in the feature branch
- **Condition**: The lambda config and script must be in `lambda/**`
- **Commit:** Commit and push the changes, open a PR from the feature branch to `integration`
- Follow the [MikMak standard release process](https://github.com/Swaven/release-process-starter-kit/tree/main?tab=readme-ov-file#release-process-starter-kit)

### 2. Deployment

- **Trigger**: When a PR targeting the `integration` branch is merged
- **Condition**: The lambda config and script must be in `lambda/**`
- **Environment**: QA (suffix: `qa`)

---

## Environment Mapping

The workflow maps Git branches to AWS environments:

| Branch Name | Environment Suffix | IAM Role Name | Config File |
| --- | --- | --- | --- |
| `integration` | `qa` | `mmm_qa_lambda_exec` | [`config.qa](http://config.qa).json` |
| `staging` | `stg` | `mmm_stg_lambda_exec` | `config.stg.json` |
| `main` | `prod` | `mmm_prod_lambda_exec` | [`config.prod](http://config.prod).json` |

### Lambda Function Naming Convention

Lambda functions are named using the pattern: `{folder-name}-{suffix}`.

**Examples:**

- `lambda/demo-python/` → `demo-python-qa` (when deployed from `integration` branch)
- `lambda/mmm-get-client-data/` → `mmm-get-client-data-stg` (when deployed from `staging` branch)

---

## Lambda Folder Structure

Each Lambda function must be in its own folder under `lambda/`. The folder structure supports both Python and Node.js runtimes.

### Required Files

#### For Python Lambdas

```
lambda/
└── {lambda-name}/
    ├── index.py (or lambda_function.py)
    ├── config.qa.json
    ├── config.stg.json
    ├── config.prod.json
    └── requirements.txt (optional)
```

#### For Node.js Lambdas

```
lambda/
└── {lambda-name}/
    ├── index.js
    ├── package.json
    ├── package-lock.json
    ├── config.qa.json
    ├── config.stg.json
    └── config.prod.json
```

### File Usage

#### `index.py` / `index.js` / `lambda_function.py`

- **Purpose**: Main Lambda handler code
- **Required**: Yes
- **Handler Format**:
    - Python: `{filename}.{function_name}` (for example, `index.handler`)
    - Node.js: `{filename}.{exported_function}` (for example, `index.handler`)

#### `package.json` (Node.js only)

- **Purpose**: Defines Node.js dependencies and project metadata
- **Required**: Yes for Node.js Lambdas
- **Usage**: The workflow runs `npm ci --production` to install dependencies

#### `package-lock.json` (Node.js only)

- **Purpose**: Locks dependency versions for reproducible builds
- **Required**: Recommended for Node.js Lambdas
- **Usage**: Used by `npm ci` to ensure consistent dependency versions

#### `requirements.txt` (Python only)

- **Purpose**: Lists Python package dependencies
- **Required**: Optional (only if external packages are needed)
- **Usage**: The workflow runs `pip install --target . -r requirements.txt`

#### `config.{env}.json`

- **Purpose**: Environment-specific Lambda configuration
- **Required**: Yes (at least one environment config)
- **Usage**: See Configuration Files section below

---

## Configuration Files

Each Lambda function requires environment-specific configuration files: [`config.qa](http://config.qa).json`, `config.stg.json`, and/or [`config.prod](http://config.prod).json`.

### Configuration File Properties

#### `handler` (string, required)

- **Description**: The Lambda handler function entry point
- **Format**: `{filename}.{function_name}`
- **Examples**:
    - Python: `"index.handler"`, `"lambda_function.lambda_handler"`
    - Node.js: `"index.handler"`, `"handler.main"`
- **Default**: `"lambda_function.lambda_handler"` (if not specified)

#### `runtime` (string, required)

- **Description**: AWS Lambda runtime identifier
- **Valid Values**:
    - Python: `"python3.11"`, `"python3.10"`, `"python3.9"`
    - Node.js: `"nodejs20.x"`, `"nodejs18.x"`, `"nodejs16.x"`
- **Default**: `"python3.11"` (if not specified)

#### `timeout` (integer, optional)

- **Description**: Maximum execution time in seconds
- **Range**: 1–900 seconds (15 minutes)
- **Default**: `60` seconds (if not specified)
- **Example**: `90`

#### `memory_size` (integer, optional)

- **Description**: Amount of memory allocated to the Lambda function in MB
- **Range**: 128–10240 MB (in 64 MB increments)
- **Default**: `128` MB (if not specified)
- **Example**: `256`

#### `role` (string, optional)

- **Description**: IAM role name (without ARN) for the Lambda function
- **Usage**: If specified, the workflow will use this role. Otherwise, it uses the default role based on the branch.
- **Example**: `"mmm_qa_lambda_exec"`
- **Note**: The role must exist in AWS or be created manually. The workflow can create default roles but not custom roles.

#### `environment_variables` (object, optional)

- **Description**: Key-value pairs of environment variables accessible to the Lambda function
- **Format**: JSON object with string keys and string values
- **Filtering**: Keys or values starting with `#` are treated as comments and ignored

**Example:**

```json
{
  "environment_variables": {
    "ENVIRONMENT": "qa",
    "TABLE_NAME": "my-table-qa",
    "#COMMENTED_VAR": "ignored"
  }
}
```

### Configuration File Examples

#### Example 1: Python Lambda (`demo-python/config.qa.json`)

```json
{
  "handler": "index.handler",
  "timeout": 90,
  "runtime": "python3.11",
  "memory_size": 256,
  "role": "mmm_qa_lambda_exec",
  "environment_variables": {
    "ENVIRONMENT": "qa"
  }
}
```

#### Example 2: Node.js Lambda (`mmm-get-client-data/config.stg.json`)

```json
{
  "handler": "index.handler",
  "timeout": 90,
  "runtime": "nodejs20.x",
  "memory_size": 256,
  "role": "mmm_stg_lambda_exec",
  "environment_variables": {
    "ENVIRONMENT": "stg",
    "CLIENT_METADATA_TABLE_NAME": "client-metadata-stg",
    "PIPELINE_INFOS_TABLE_NAME": "pipeline-infos-stg"
  }
}
```

---

## Deployment Process

The workflow follows these steps:

### 1. Validation Phase

- Checks AWS connectivity and permissions
- Lists all Lambda folders in `lambda/` directory
- Fetches existing Lambda functions from AWS (matching the environment suffix)
- Identifies which Lambdas exist in AWS vs which are new

### 2. Change Detection Phase

- For **push events**: Compares current commit with previous commit
- For **PR merge events**: Compares PR base with PR head
- For **manual triggers**: Updates all existing Lambdas
- Only Lambdas with changes in their folder are marked for update

### 3. IAM Setup Phase

- Checks if IAM role exists (based on branch name)
- Creates role if it does not exist
- Attaches `AWSLambdaBasicExecutionRole` policy
- Retrieves role ARN for Lambda creation

### 4. Update Phase (for existing Lambdas)

- Reads configuration from the appropriate `config.{env}.json` file
- Installs dependencies (`npm ci` for Node.js, `pip install` for Python)
- Creates deployment package (ZIP file)
- Updates Lambda function configuration (handler, runtime, timeout, memory, env vars)
- Updates Lambda function code
- Publishes new version with deployment metadata

### 5. Create Phase (for new Lambdas)

- Reads configuration from the appropriate `config.{env}.json` file
- Installs dependencies
- Creates deployment package (ZIP file)
- Creates Lambda function in AWS
- Publishes initial version

### 6. Cleanup Phase

- Removes temporary ZIP files
- Lists all processed Lambda functions

---

## Examples

### Example 1: Python Lambda (`demo-python`)

**Folder Structure:**

```
lambda/demo-python/
├── index.py
├── config.qa.json
└── config.stg.json
```

**Handler Code (`index.py`):**

```python
import json

def handler(event, context):
    response = {
        "message": "Hello from AWS Lambda (Python)!",
        "input_event": event,
        "request_id": [context.aws](http://context.aws)_request_id if context else None,
    }

    return {
        "statusCode": 200,
        "headers": {"Content-Type": "application/json"},
        "body": json.dumps(response),
    }
```

**Configuration (`config.qa.json`):**

```json
{
  "handler": "index.handler",
  "timeout": 90,
  "runtime": "python3.11",
  "memory_size": 256,
  "role": "mmm_qa_lambda_exec",
  "environment_variables": {
    "ENVIRONMENT": "qa"
  }
}
```

**Deployment:**

- When merged to `integration` branch → Creates/updates `demo-python-qa`
- When merged to `staging` branch → Creates/updates `demo-python-stg`

---

### Example 2: Node.js Lambda (`mmm-get-client-data`)

**Folder Structure:**

```
lambda/mmm-get-client-data/
├── index.js
├── package.json
├── package-lock.json
├── config.qa.json
├── config.stg.json
├── config.prod.json
└── README.md
```

**Handler Code (`index.js`):**

- Exports `handler` function that processes API Gateway events
- Queries DynamoDB tables using AWS SDK v3
- Returns JSON response with client data

**Dependencies (`package.json`):**

```json
{
  "dependencies": {
    "@aws-sdk/client-dynamodb": "^3.490.0",
    "@aws-sdk/lib-dynamodb": "^3.490.0"
  }
}
```

**Configuration (`config.stg.json`):**

```json
{
  "handler": "index.handler",
  "timeout": 90,
  "runtime": "nodejs20.x",
  "memory_size": 256,
  "role": "mmm_stg_lambda_exec",
  "environment_variables": {
    "ENVIRONMENT": "stg",
    "CLIENT_METADATA_TABLE_NAME": "client-metadata-stg",
    "PIPELINE_INFOS_TABLE_NAME": "pipeline-infos-stg"
  }
}
```

**Deployment:**

- When merged to `integration` branch → Creates/updates `mmm-get-client-data-qa`
- When merged to `staging` branch → Creates/updates `mmm-get-client-data-stg`
- When merged to `main` branch → Creates/updates `mmm-get-client-data-prod`

---

## Best Practices

### 1. Environment-Specific Configuration

- Always create separate config files for each environment (`config.qa.json`, `config.stg.json`, [`config.prod](http://config.prod).json`).
- Use environment variables for environment-specific values (table names, API endpoints, and so on).

### 2. Dependency Management

- For Node.js: Always commit `package-lock.json` for reproducible builds.
- For Python: Pin versions in `requirements.txt` (for example, `boto3==1.28.0`).

### 3. Handler Function

- Python: Use `def handler(event, context):` as the function signature.
- Node.js: Use `exports.handler = async (event, context) => {}` or `module.exports.handler = ...`.

### 4. Testing Before Deployment

- Test Lambda functions locally before pushing changes.
- Use AWS SAM or LocalStack for local testing.
- Verify configuration files are valid JSON.

### 5. IAM Roles

- Use the default role naming convention (`mmm_{env}_lambda_exec`) when possible.
- If custom roles are needed, create them manually in AWS before deployment.
- Ensure roles have necessary permissions (DynamoDB, S3, and so on).

### 6. Change Detection

- The workflow only deploys Lambdas with changes in their folder.
- To force the deployment of all Lambdas, use manual workflow dispatch.
- Changes to config files trigger deployment.

---

## Troubleshooting

### Lambda Not Deploying

- **Check**: Ensure changes are in the `lambda/{function-name}/` folder.
- **Check**: Verify the workflow is triggered (check GitHub Actions tab).
- **Check**: Review workflow logs for errors.

### Configuration Errors

- **Check**: Ensure config files are valid JSON.
- **Check**: Verify handler path matches actual file/function name.
- **Check**: Ensure runtime matches the code language.

### IAM Permission Errors

- **Check**: Verify AWS credentials have Lambda and IAM permissions.
- **Check**: Ensure IAM role exists or can be created.
- **Check**: Verify role has necessary policies attached.

### Dependency Installation Failures

- **Check**: Verify `package.json` or `requirements.txt` syntax.
- **Check**: Ensure all dependencies are available and compatible.
- **Check**: Review workflow logs for specific package errors.

### Function Update Failures

- **Check**: Verify Lambda function exists in AWS.
- **Check**: Ensure function is in `Active` state before updating.
- **Check**: Review AWS CloudWatch logs for runtime errors.

---

## Additional Resources

- [AWS Lambda Documentation](https://docs.aws.amazon.com/lambda/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [AWS SDK for Python (Boto3)](https://boto3.amazonaws.com/v1/documentation/api/latest/index.html)
- [AWS SDK for JavaScript v3](https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/)
