# Amazon Web Services

!!! note ""
    This page is based on the official documentation [https://www.cloudcarbonfootprint.org/docs/aws](https://www.cloudcarbonfootprint.org/docs/aws).
    <br />
    You can find here **more details and real examples**.

This page details how to configure the AWS integration for the Cloud Carbon Footprint tool.

## Configuration

*The hereby steps are retrieved from the official documentation and explained.*

Steps to follow:

1. [Create S3 Bucket for Carbon Footprint data](#create-s3-bucket-for-carbon-footprint-data)
1. [Ensure your aws account has the correct permissions](#ensure-your-aws-account-has-the-correct-permissions)
1. [Enable the Cost and Usage Billing AWS feature](#enable-the-cost-and-usage-billing-aws-feature)
1. [Setup Athena DB to save the Cost and Usage Reports](#setup-athena-db-to-save-the-cost-and-usage-reports)
1. [Configure aws credentials locally, using awscli](#configure-aws-credentials-locally-using-awscli)
1. [Configure environmental variables for the api and client](#configure-environmental-variables-for-the-api-and-client)

### Create S3 Bucket for Carbon Footprint data

```bash
aws s3api create-bucket \
    --bucket cloudia-ccf-billing-data \
    --region eu-west-3 \
    --create-bucket-configuration LocationConstraint=eu-west-3 \
    --profile aws_cloudia_root_admin
```

??? note "Output"

    ```json
    {
        "Location": "http://cloudia-ccf-billing-data.s3.amazonaws.com/"
    }
    ```

```bash
aws s3api create-bucket \
    --bucket cloudia-ccf-queryresult-data \
    --region eu-west-3 \
    --create-bucket-configuration LocationConstraint=eu-west-3 \
    --profile aws_cloudia_root_admin
```

??? note "Output"

    ```json
    {
        "Location": "http://cloudia-ccf-queryresult-data.s3.amazonaws.com/"
    }
    ```


#### Configure policy for Bucket

Configure the bucket `cloudia-ccf-billing-data` with policy to allow the AWS Cost and Usage Report (CUR) to write data.

* **Root account ID**: `123456789012` *(computed from the previous step)*
* **S3 Bucket Billing**: `cloudia-ccf-billing-data` *(computed from the previous step)*

```json title="aws_cur_bucket_policy.json"
{
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "billingreports.amazonaws.com"
            },
            "Action": [
                "s3:GetBucketAcl",
                "s3:GetBucketPolicy"
            ],
            "Resource":"arn:aws:s3:::cloudia-ccf-billing-data",
            "Condition": {
                "StringEquals": {
                    "aws:SourceArn": "arn:aws:cur:us-east-1:123456789012:definition/*",
                    "aws:SourceAccount": "123456789012"
                }
            }
        },
        {
            "Sid": "Stmt1335892526596",
            "Effect": "Allow",
            "Principal": {
                "Service": "billingreports.amazonaws.com"
            },
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::cloudia-ccf-billing-data/*",
            "Condition": {
                "StringEquals": {
                    "aws:SourceArn": "arn:aws:cur:us-east-1:123456789012:definition/*",
                    "aws:SourceAccount": "123456789012"
                }
            }
        }
    ]
}
```

```bash
aws s3api put-bucket-policy \
    --bucket cloudia-ccf-billing-data \
    --policy file://aws_cur_bucket_policy.json \
    --profile aws_cloudia_root_admin
```

### Ensure your aws account has the correct permissions

Cloud Carbon Footprint gives a AWS CloudFormation template to create the necessary resources: [https://github.com/cloud-carbon-footprint/cloud-carbon-footprint/blob/trunk/cloudformation/ccf-app.yaml](https://github.com/cloud-carbon-footprint/cloud-carbon-footprint/blob/trunk/cloudformation/ccf-app.yaml).

The CloudFormatipon Template takes some input parameters.

| Parameter | Description | Default value | Example |
| --- | --- | --- | --- |
| AuthorizedRole | The ARN of the role that will be authorized to access the data | `arn:aws:iam::<ACCOUNT ID>:<NAME>` | `arn:aws:iam::123456789012:user/cloudia-svc` |
| BillingDataBucket | The ARN of the S3 bucket where the Cost and Usage Reports will be saved | `arn:aws:s3:::<YOUR BUCKET NAME>` | `arn:aws:s3:::cloudia-ccf-billing-data` |
| QueryResultsBucket | The name of the S3 bucket where the Athena query results will be saved | `arn:aws:s3:::<YOUR BUCKET NAME>` | `arn:aws:s3:::cloudia-ccf-queryresult-data` |

```bash
curl -O https://raw.githubusercontent.com/cloud-carbon-footprint/cloud-carbon-footprint/trunk/cloudformation/ccf-app.yaml
aws cloudformation create-stack \
    --stack-name ccf-app \
    --template-body file://ccf-app.yaml \
    --profile aws_cloudia_root_admin \
    --region eu-west-3 \
    --capabilities "CAPABILITY_NAMED_IAM" \
    --parameters \
        "ParameterKey=AuthorizedRole,ParameterValue=arn:aws:iam::123456789012:user/cloudia-svc" \
        "ParameterKey=BillingDataBucket,ParameterValue=arn:aws:s3:::cloudia-ccf-billing-data" \
        "ParameterKey=QueryResultsBucket,ParameterValue=arn:aws:s3:::cloudia-ccf-queryresult-data"
```

Links:

* [AWS CLI Documentation - create-stack](https://docs.aws.amazon.com/cli/latest/reference/cloudformation/create-stack.html)

### Enable the Cost and Usage Billing AWS feature (legacy CUR)

To enable the Cost and Usage Billing feature, you need to create a Cost and Usage Report (CUR). You can create a CUR using the [AWS Management Console](https://us-east-1.console.aws.amazon.com/billing/home?region=us-east-1#/reports) ([AWS Documentation](https://docs.aws.amazon.com/cur/latest/userguide/cur-create.html)) or the AWS CLI as following.

To create a CUR using the AWS CLI, you need to create a JSON file (here named `report-definition.json`) and at least replace the following attributes:

| Attribute | Description | Default value | Example |
| --- | --- | --- | --- |
| `S3Bucket` | The name of the S3 bucket where to write the report data | `<YOUR BUCKET NAME>` | `cloudia-ccf-billing-data` |
| `S3Prefix` | The prefix of the S3 bucket | `<BUCKET PREFIX>` | `cloudia` |
| `S3Region` | The region of the S3 bucket | `<BUCKET REGION>` | `eu-west-3` |

For more information about the CLI command, see the [AWS CLI Documentation - put-report-definition](https://docs.aws.amazon.com/cli/latest/reference/cur/put-report-definition.html).

```json title="aws_cur_report_definition.json"
{
    "ReportName": "CloudiaReport",
    "TimeUnit": "DAILY",
    "Format": "Parquet",
    "Compression": "Parquet",
    "AdditionalSchemaElements": [
        "RESOURCES"
    ],
    "S3Bucket": "cloudia-ccf-billing-data",
    "S3Prefix": "cloudia",
    "S3Region": "eu-west-3",
    "AdditionalArtifacts": [
        "ATHENA"
    ],
    "RefreshClosedReports": true,
    "ReportVersioning": "OVERWRITE_REPORT"
}
```

```bash
aws cur put-report-definition --report-definition file://aws_cur_report_definition.json --profile aws_cloudia_root_admin --region us-east-1
```

Once the report definition is created, the refreshness is realized several times a day, you might need to wait a few hours before the first report is available.

Links:

* [AWS Documentation - CUR: What is CUR](https://docs.aws.amazon.com/cur/latest/userguide/what-is-cur.html)
* [AWS Documentation - CUR: Creating Cost and Usage Reports](https://docs.aws.amazon.com/cur/latest/userguide/cur-create.html)
* [AWS CLI Documentation - cur: put-report-definition](https://docs.aws.amazon.com/cli/latest/reference/cur/put-report-definition.html)

### Setup Athena DB to save the Cost and Usage Reports

Once the data is refreshed in the S3 bucket, you can create an Athena DB to query the data.

First, create the Database `ccf` in Athena.

```bash
aws athena start-query-execution --query-string "CREATE DATABASE ccf" --profile aws_cloudia_root_admin --region eu-west-3

```

Then, create the table in Athena. The CUR provides a SQL script to create the Athena Table inside the data bucket. It is located in your bucket `<S3Bucket>` at `<S3Prefix>/<ReportName>/<YYYYMMDD>-<YYYYMMDD>`.

Download the script (should be named `ccf-create-table.sql`) and execute it in Athena.

```bash
athena_query=$(cat ./ccf-create-table.sql | tr '\n' ' ')
aws athena start-query-execution --query-string "${athena_query}" --query-execution-context Database=ccf --profile aws_cloudia_root_admin --region eu-west-3
```

??? warning "An error occurred (InvalidRequestException) when calling the StartQueryExecution operation: line 1:8: mismatched input 'EXTERNAL'."
    The following error might occur:
    <br />
    ```raw
    An error occurred (InvalidRequestException) when calling the StartQueryExecution operation: line 1:8: mismatched input 'EXTERNAL'. Expecting: 'MATERIALIZED', 'MULTI', 'OR', 'PROTECTED', 'ROLE', 'SCHEMA', 'TABLE', 'VIEW'
    ```
    It might mean that the object names are not well defined. This happend when using hyphens (`-`) in database and/or table names.
    <br />
    **Edit the script** and **add backticks** (`` ` ``) around the database/table name: from `ccf-app` to `` `ccf-app` ``.

When the database and the table are created, you can query the data using the Athena console or the AWS CLI.

???+ info "Force load the Athena Table Partitions"
    The Athena query might return no data. You can force refresh the partitions by clicking on button `Load partitions`.

Links:

* [AWS Documentation - CUR: Creating an Athena table](https://docs.aws.amazon.com/cur/latest/userguide/create-manual-table.html)
* [AWS Documentation - Athena: Create Database](https://docs.aws.amazon.com/athena/latest/ug/create-database.html)
* [AWS Documentation - Athena: Load Partitions](https://docs.aws.amazon.com/cur/latest/userguide/upload-report-partitions.html)
* [AWS CLI - Athena: start-query-execution](https://docs.aws.amazon.com/cli/latest/reference/athena/start-query-execution.html)

### Configure aws credentials locally, using awscli

### Configure environmental variables for the api and client

Clone the Cloud Carbon Footprint repository.

```bash
git clone https://github.com/cloud-carbon-footprint/cloud-carbon-footprint.git
```

Update the `.env` files for the API and the Client.

!!! note ""
    The `.env` files are well documented and provide examples.

#### Configure the Client

Copy the `.env.template` and update the `packages/client/.env` environment variables files for the client.

```bash
cp packages/client/.env.template packages/client/.env
vi packages/client/.env
```

#### Configure the API

Update the `packages/api/.env` environment variables files for the API..

```bash
cp packages/api/.env.template packages/api/.env
vi packages/api/.env
```

???+ info
    You can disable some Cloud Providers on commenting their related variables:

    * `AWS_` prefix for Amazon Web Services
    * `GCP_` prefix for Google Cloud
    * `AZURE_` prefix for Microsoft Azure

## Execution

### Docker Compose

Cloud Carbon Footprint provides a `docker-compose.yml` file to run the application locally.

```bash
curl -O https://raw.githubusercontent.com/cloud-carbon-footprint/cloud-carbon-footprint/trunk/docker-compose.yml
```

The `docker-compose.yml` file requires to define the secret variables into files.


Here is the list of the variables and the corresponding secret files:

| Variable | Secret file |
| --- | --- |
| `AWS_TARGET_ACCOUNT_ROLE_NAME` | `~/.docker/secrets/AWS_TARGET_ACCOUNT_ROLE_NAME` |
| `AWS_ATHENA_DB_NAME` | `~/.docker/secrets/AWS_ATHENA_DB_NAME` |
| `AWS_ATHENA_DB_TABLE` | `~/.docker/secrets/AWS_ATHENA_DB_TABLE` |
| `AWS_ATHENA_REGION` | `~/.docker/secrets/AWS_ATHENA_REGION` |
| `AWS_ATHENA_QUERY_RESULT_LOCATION` | `~/.docker/secrets/AWS_ATHENA_QUERY_RESULT_LOCATION` |
| `AWS_BILLING_ACCOUNT_ID` | `~/.docker/secrets/AWS_BILLING_ACCOUNT_ID` |
| `AWS_BILLING_ACCOUNT_NAME` | `~/.docker/secrets/AWS_BILLING_ACCOUNT_NAME` |
| `GCP_BIG_QUERY_TABLE` | `~/.docker/secrets/GCP_BIG_QUERY_TABLE` |
| `GCP_BILLING_PROJECT_ID` | `~/.docker/secrets/GCP_BILLING_PROJECT_ID` |
| `GCP_BILLING_PROJECT_NAME` | `~/.docker/secrets/GCP_BILLING_PROJECT_NAME` |
| `AZURE_CLIENT_ID` | `~/.docker/secrets/AZURE_CLIENT_ID` |
| `AZURE_CLIENT_SECRET` | `~/.docker/secrets/AZURE_CLIENT_SECRET` |
| `AZURE_TENANT_ID` | `~/.docker/secrets/AZURE_TENANT_ID` |

Here is the list of the variables and the corresponding content:

| Variable | Description | Example |
| --- | --- | --- |
| **Amazon Web Services** | | |
| `AWS_TARGET_ACCOUNT_ROLE_NAME` | The ARN of the role that will be authorized to access the data | `arn:aws:iam::123456789012:user/admin` |
| `AWS_ATHENA_DB_NAME` | The name of the Athena DB | `ccf` |
| `AWS_ATHENA_DB_TABLE` | The name of the Athena Table | `ccf-app` |
| `AWS_ATHENA_REGION` | The region of the Athena DB | `eu-west-3` |
| `AWS_ATHENA_QUERY_RESULT_LOCATION` | The location of the Athena query results | `s3://cloudia-ccf-queryresult-data` |
| `AWS_BILLING_ACCOUNT_ID` | The ID of the billing account | `123456789012` |
| `AWS_BILLING_ACCOUNT_NAME` | The name of the billing account | `cloudia` |
| **Google Cloud** | | |
| `GCP_BIG_QUERY_TABLE` | The name of the Big Query Table | `cloudia` |
| `GCP_BILLING_PROJECT_ID` | The ID of the billing project | `cloudia` |
| `GCP_BILLING_PROJECT_NAME` | The name of the billing project | `cloudia` |
| **Microsoft Azure** | | |
| `AZURE_CLIENT_ID` | The ID of the client | `123456789012` |
| `AZURE_CLIENT_SECRET` | The secret of the client | `cloudia` |
| `AZURE_TENANT_ID` | The ID of the tenant | `cloudia` |

```bash
docker compose -f docker-compose.yml up
```

### Troubleshooting

Running the application with Docker might ends with several issues. Here are a quick list of the most common ones.

**The website is good but there's no data**

The nginx engine mounts the configuration file `nginx.conf` from the host directory `./docker/nginx.conf`.

The nginx configuration file set the `proxy_pass` directive with the internal DNS of the API: `proxy_pass http://api:4000$request_uri;`. The `api` might not be resolved by the container. If so, you may need to change the DNS with the IP address of the container running the API.

Retrieve the IP address of the container running the API:

```bash
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' cloud-carbon-footprint-api-1
```

And update the mounter `./docker/nginx.conf` file with the IP address.

```{ .diff }
events {
}

http {
    server {
        listen 80;
        include /etc/nginx/mime.types;
        root /var/www;
        index index.html index.htm;
        location /api {
            resolver 127.0.0.11;
            proxy_set_header X-Forwarded-Host $host;
            proxy_set_header X-Forwarded-Server $host;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
-            proxy_pass http://api:4000$request_uri;
+            proxy_pass http://172.24.0.2:4000$request_uri;

        }
        location / {
            try_files $uri $uri/ =404;
        }
    }
}
```

## Links

* Official documentation: [https://www.cloudcarbonfootprint.org/docs/aws](https://www.cloudcarbonfootprint.org/docs/aws)
