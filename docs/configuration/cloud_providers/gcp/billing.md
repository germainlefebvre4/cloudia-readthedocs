# GCP Billing

## Principles

Retrieving Cloud Projects Billing requires actions on specified Cloud Projects. You need to provide the platform with the credentials to access the Cloud Provider dedicated to the Billing scope in order to retrieve the billing data.

## Configuration

You have set up the Cloud Provider Access explained in page [GCP Projects](projects.md) section Google Cloud.

Requirements:

* Google Cloud Organization ID
* Google Cloud Project Name
* Email of the Service Account created on the organization

Values that will be used in the following configuration:

* Google Cloud Organization ID: `123456789012`
* Google Cloud Billing Account ID: `123456_123ABC_1A2B3C`
* Service Account Name: `cloudia-svc`
* Service Account Key File: `gcp_service-account_key-file.json`
* Service Account Email: `cloudia-svc@cloudia-billing-prod.iam.gserviceaccount.com`

### Billing export project

Create or select a Google Cloud project to store the billing data.

```bash
gcloud config set project cloudia-billing-prod
```

Create the dataset `billing_export` in the project `cloudia-billing-prod`.

```bash
gcloud alpha bq datasets create billing_export --description 'Organization Billing Export Dataset'
```

### Grant Service Account access to BigQuery billing dataset

Grant the service account based on the email (i.e. `cloudia-svc@cloudia-billing-prod.iam.gserviceaccount.com`) the `BigQuery Data Viewer` role on the project `cloudia-billing-prod`.

```bash
gcloud projects add-iam-policy-binding cloudia-billing-prod --member="serviceAccount:cloudia-svc@cloudia-project.iam.gserviceaccount.com" --role="roles/bigquery.dataViewer"
```

### Configure the billing export on the organization

Here is the sumup of actions described in the official documentation [Export Billing Data to a BigQuery Dataset](https://cloud.google.com/billing/docs/how-to/export-data-bigquery-setup).

Go at the [Billing export page](https://console.cloud.google.com/billing/export) and click on `Configure billing export`.

In the section **Standard usage cost** click on "Edit Settings" and set the following values:

* Project name: `cloudia-billing-prod`
* Dataset: `billing_export`

THe billing export is configured and will happen every 24 hours.
