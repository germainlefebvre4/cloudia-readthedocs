---
hide:
    - toc
---
# Environment Variables

## API Configuration

### Environment variables

The platform is configured through environment variables. The environment variables are defined in the `.env` file.

| Name | Description | Default value |
| --- | --- | --- |
| `BACKEND_CORS_ORIGINS` | List of allowed origins for the CORS headers | `["http://localhost", "http://localhost:3000", "http://localhost:8080", "https://localhost", "https://localhost:3000", "https://localhost:8080"]` |
| **Google Cloud Platform** | | |
| `GCP_SERVICE_ACCOUNT_JSON_KEY_FILE` | Path to the Google Cloud Platform service account JSON key file | `gcp_service-account_key-file.json` |
| `GCP_ORGANIZATION_ID` | Google Cloud Platform organization ID | `123456789012` |
| `GCP_BILLING_ACCOUNT_ID` | Google Cloud Platform billing account ID | `123456_123ABC_1A2B3C` |
| `GCP_BILLING_EXPORT_PROJECT_ID` | Google Cloud Platform billing export project ID | `cloudia-billing-prod` |
| `GCP_BILLING_EXPORT_DATASET_NAME` | Google Cloud Platform billing export dataset name | `billing_export` |
| **Amazon Web Services** | ||
| `AWS_ACCESS_KEY_ID` | Amazon Web Services access key ID | `xxxxxxxxxxxxxxxxxxxx` |
| `AWS_SECRET_ACCESS_KEY` | Amazon Web Services secret access key ID | `xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx` |
| `AWS_DEFAULT_REGION` | Amazon Web Services default region | `us-east-1` |
