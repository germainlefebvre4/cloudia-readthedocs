# GCP Service Account

## Principles

The IAM Service Account will allow retrieving information from GCP using the API of the Cloud Provider.

## Configuration

Requirements:

* Google Cloud Organization ID
* Google Cloud Project Name

Values that will be used in the following configuration:

* Google Cloud Organization ID: `123456789012`
* Google Project Name: `cloudia-project`
* Service Account Name: `cloudia-svc`
* Service Account Email: `cloudia-svc@cloudia-project.iam.gserviceaccount.com` (mind to replace with your own)
* Service Account Key File: `gcp_service-account_key-file.json`

Choose a Google Cloud Platform project to store the assets relating to Cloudia.

```bash
gcloud config set project cloudia-project
```

### Create IAM Service Account and generate credentials

Create a service account.

```bash
gcloud iam service-accounts create cloudia-svc --description="DESCRIPTION" --display-name="resource-manager-list"
```

Create the service account key file.

```bash
gcloud iam service-accounts keys create gcp_service-account_key-file.json --iam-account cloudia-svc@cloudia-project.iam.gserviceaccount.com
```

The service account key file `gcp_service-account_key-file.json` will be used in the API through the `GCP_SERVICE_ACCOUNT_JSON_KEY_FILE` environment variable.
