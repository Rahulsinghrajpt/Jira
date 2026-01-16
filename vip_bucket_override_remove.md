# VIP Bucket Override Removal

## Goal
Remove the custom S3 bucket override logic from the data ingestion pipeline and strictly enforce the standard naming convention: `mmm-{env}-data-{client_id}`. This simplifies the codebase and ensures environment consistency.

## Current Behavior (Baseline)
- The `data_transfer` Lambda contains a `VIP_BUCKET_OVERRIDES` map for hardcoded overrides.
- The `get_vip_bucket_name` function supports an `override_bucket` parameter from Step Function events or DynamoDB.
- A helper function `_normalize_s3_bucket_name` processes these overrides.

## Target Behavior
- **Strict Standardization**: All data transfers will target `mmm-{env}-data-{client_id}`.
- **Code Simplification**: Removal of approximately 100+ lines of override-related logic across Lambda 1 (`get-client`) and Lambda 2 (`data-transfer`).
- **Configuration Driven**: The bucket prefix is resolved dynamically via the `ENVIRONMENT` environment variable.

## Implementation Details

### 1) Data Transfer Lambda (`lambda/data-transfer/lambda_function.py`)

#### [DELETE] Global Overrides & Helpers
- Remove `VIP_BUCKET_OVERRIDES` dictionary.
- Remove `VAR_VIP_BUCKET` (if any hardcoded references exist).
- Remove `_normalize_s3_bucket_name()` helper function.

#### [REFACTOR] Standardized Bucket Resolution
The resolution logic will be simplified to a single pattern.
```python
def get_vip_bucket_name(client_id: str) -> str:
    """Compute VIP bucket name: mmm-{env}-data-{client_id}"""
    # Hyphen (-) is used for S3 compatibility
    if CONFIG_AVAILABLE:
        return config.get_vip_bucket_name(client_id)
    return f"{VIP_BUCKET_PREFIX}-{client_id}"
```

#### [MODIFY] Signature Cleanup
- **`upload_to_client_bucket(...)`**: Remove the `destination_bucket` argument. Internalize bucket resolution.
```python
def upload_to_client_bucket(
    file_data: bytes,
    source_key: str,
    client_id: str,
    brand_name: str,
    retailer_id: str
) -> str:
    # Compute VIP bucket name dynamically
    vip_bucket = get_vip_bucket_name(client_id)
    # ... rest of upload logic ...
```
- **`process_file(...)`**: Remove `destination_bucket` parameter.
- **`lambda_handler`**: Remove any logic that extracts `override_bucket` or `destination_bucket` from the event.

### 2) Get Client Lambda (`lambda/get-client/lambda_function.py`)
- Remove logic that extracts `destination_bucket` from DynamoDB `pipeline_info` records.
- Clean up the output payload sent to the Step Function to remove the `destination_bucket` key.

## Testing Plan

### A) Unit Tests
- Update tests to ensure `get_vip_bucket_name` returns correctly for `dev`, `qa`, and `prod` environments.

### B) Integration Validation
- Verify that data is written to `mmm-{env}-data-{client_id}` even if a `destination_bucket` was previously set in DynamoDB.

## Acceptance Criteria
- Code is free of `VIP_BUCKET_OVERRIDES` and `destination_bucket` variables.
- S3 uploads consistently follow the `mmm-{env}-data-{client_id}` pattern.
