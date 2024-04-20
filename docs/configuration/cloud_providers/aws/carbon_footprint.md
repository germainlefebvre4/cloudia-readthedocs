# AWS Carbon Footprint

## Principles

Retrieving Cloud Projects Carbon Footprint requires actions on specified Cloud Projects. Actions can be realized directly from the root account which has access to all the projects through AWS Cloudformation StackSet.

## Requirements

The following actions are realized from the root account based on a user with Administrator privileges connected to the AWS CLI.

Several informations require to be retrieved. Check these values before starting the configuration.

### IAM User used as Service Account

Follow the dedicated page at [AWS Accounts](projects.md) in the section Amazon Web Services.

### Root Account ID

Retrieve the root account ID from the AWS CLI:

```bash
aws sts get-caller-identity --query "Account" --output text
```

### Root Account Organization ID

Retrieve the root account Organization ID from the AWS CLI:

```bash
aws organizations list-roots | jq -r '.Roots[0].Id'
```

## Configuration

### Create S3 Bucket for Carbon Footprint data

```bash
aws s3api create-bucket --bucket cloudia-ccf-billing-data --region eu-west-3 --create-bucket-configuration LocationConstraint=eu-west-3
```

??? note "Output"

    ```json
    {
        "Location": "http://cloudia-ccf-billing-data.s3.amazonaws.com/"
    }
    ```

```bash
aws s3api create-bucket --bucket cloudia-ccf-queryresult-data --region eu-west-3 --create-bucket-configuration LocationConstraint=eu-west-3
```

??? note "Output"

    ```json
    {
        "Location": "http://cloudia-ccf-queryresult-data.s3.amazonaws.com/"
    }
    ```

### Create the IAM Role for reading Carbon Footprint data

* **Root account ID**: `123456789012` *(computed from the previous step)*
* **IAM User name**: `cloudia-svc` *(required from the previous step)*
* **Role name**: `cloudia-role`
* **S3 Bucket for data**: `cloudia-ccf-billing-data` *(computed from the previous step)*
* **S3 Bucket for query results**: `cloudia-ccf-queryresult-data` *(computed from the previous step)*

Here we go with the awscli commands.

```yaml title="cloudia-role.yaml"
Parameters:
  AuthorizedRole:
    Description: The trusted entity's ARN that can assume the CCF role
    Type: String
    Default: arn:aws:iam::<ACCOUNT ID>:<NAME> # Will be replaced by the AWS CLI --parameters argument
  BillingDataBucket:
    Description: The S3 bucket where the CUR data lives
    Type: String
    Default: arn:aws:s3:::<YOUR BUCKET NAME> # Will be replaced by the AWS CLI --parameters argument
  QueryResultsBucket:
    Description: The S3 bucket where Athena query results will be stored
    Type: String
    Default: arn:aws:s3:::<YOUR BUCKET NAME> # Will be replaced by the AWS CLI --parameters argument
Resources:
  CCFAthenaRole:
    Type: 'AWS::IAM::Role'
    Description: This role allows Cloud Carbon Footprint application to read Cost and Usage Reports via AWS Athena
    Properties:
      RoleName: 'cloudia-carbonfootprint-ccf'
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Principal:
              AWS: !Ref AuthorizedRole
            Action: sts:AssumeRole
      Policies:
        - PolicyName: athena
          PolicyDocument:
            Version: 2012-10-17
            Statement:
              - Effect: Allow
                Action:
                  - athena:StartQueryExecution
                  - athena:GetQueryExecution
                  - athena:GetQueryResults
                  - athena:GetWorkGroup
                Resource: '*'
        - PolicyName: glue
          PolicyDocument:
            Version: 2012-10-17
            Statement:
              - Effect: Allow
                Action:
                  - glue:GetDatabase
                  - glue:GetTable
                  - glue:GetPartitions
                Resource: '*'
        - PolicyName: s3
          PolicyDocument:
            Version: 2012-10-17
            Statement:
              - Effect: Allow
                Action:
                  - s3:GetBucketLocation
                  - s3:GetObject
                  - s3:ListBucket
                  - s3:ListBucketMultipartUploads
                  - s3:ListMultipartUploadParts
                  - s3:AbortMultipartUpload
                  - s3:PutObject
                Resource:
                  - !Ref BillingDataBucket
                  - !Join [ "", [ !Ref BillingDataBucket, "/*"] ]
                  - !Ref QueryResultsBucket
                  - !Join [ "", [ !Ref QueryResultsBucket, "/*"] ]
        - PolicyName: ce
          PolicyDocument:
            Version: 2012-10-17
            Statement:
              - Effect: Allow
                Action:
                  - ce:GetRightsizingRecommendation
                Resource: '*'
```

```bash
CLOUDIA_ROOT_ACCOUNT_ID=$(aws sts get-caller-identity --query "Account" --output text)
aws cloudformation create-stack \
    --stack-name cloudia-role \
    --template-body file://cloudia-role.yaml \
    --region eu-west-3 \
    --capabilities "CAPABILITY_NAMED_IAM" \
    --parameters \
        ParameterKey=AuthorizedRole,ParameterValue=arn:aws:iam::${CLOUDIA_ROOT_ACCOUNT_ID}:user/cloudia-svc \
        ParameterKey=BillingDataBucket,ParameterValue=arn:aws:s3:::cloudia-ccf-billing-data \
        ParameterKey=QueryResultsBucket,ParameterValue=arn:aws:s3:::cloudia-ccf-queryresult-data \
```

??? note "Output"

    ```json
    {
        "StackId": "arn:aws:cloudformation:eu-west-3:123456789012:stack/cloudia-role/db282e60-ff0f-11ee-aba9-0a3c90195a3d"
    }
    ```

### Grant Service Account (root account) access to Carbon Footprint data (sub-accounts)

> The Cloudformation StackSet allows to deploy a stack on all the sub-accounts directly from the root account.

On the **Root Account**, deploy the Cloud Formation template containing the IAM Role through the Cloudformation StackSet.

The IAM Role contains the policy to allow the Service Account `claudia-svc` to retrieve the Carbon Footprint data from the sub-accounts.

* **Root account ID**: `123456789012` *(computed from the previous step)*
* **Root account Organization ID**: `r-ab12` *(computed from the previous step)*
* **IAM User name**: `cloudia-svc` *(required from the previous step)*
* **Role name**: `cloudia-read-role`
* **Policy name**: `Cloudia-Read-Data-Policy`

Here we go with the awscli commands.

#### StackSet Template

```yaml title="cloudia-read-role.yaml"
---
AWSTemplateFormatVersion: "2010-09-09"
Description: Creates a stack containing an IAM Role for reading Cloudia data
Parameters:
  CloudiaReadDataRole:
    Type: String
    Default: cloudia-read-role
    Description: Role for reading Cloudia data
  ManagementAccount:
    Description: Management Account ID number
    Type: String
Resources:
  CloudiaReadAccountDataRole:
    Type: 'AWS::IAM::Role'
    Properties:
      RoleName:
        !Ref CloudiaReadDataRole
      Path: /
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              AWS:
                - !Sub 'arn:aws:iam::${ManagementAccount}:user/cloudia-svc'
            Action:
              - 'sts:AssumeRole'
      Policies:
        - PolicyName: Cloudia-Read-Data-Policy
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Sid: CloudiaReadData
                Effect: Allow
                Resource: "*"
                Action:
                  - 'sustainability:GetCarbonFootprintSummary'
```

#### Create the StackSet

Create the StackSet on the **root account** based on the file `cloudia-read-role.yaml`:

```bash
CLOUDIA_ROOT_ACCOUNT_ID=$(aws sts get-caller-identity --query "Account" --output text)
# Value would be 123456789012 in this example
aws cloudformation create-stack-set \
    --stack-set-name cloudia-stack-set \
    --description "Cloudia StackSet" \
    --template-body file://cloudia-read-role.yaml \
    --region eu-west-3 \
    --permission-model "SERVICE_MANAGED" \
    --capabilities "CAPABILITY_NAMED_IAM" \
    --auto-deployment "Enabled=true,RetainStacksOnAccountRemoval=true" \
    --managed-execution "Active=false" \
    --parameters \
        ParameterKey=ManagementAccount,ParameterValue=${CLOUDIA_ROOT_ACCOUNT_ID} \
        ParameterKey=CloudiaReadDataRole,ParameterValue=cloudia-read-role
```

??? note "Output"

    ```json
    {
        "StackSetId": "ccft-stack-set-2:4466a1aa-ca61-4933-88fa-ad65cd6ef4f3"
    }
    ```

#### Create the Stack Instances

Create the Stack Instances related to the StackSet on the **root account** based on the root account Organization ID:

```bash
CLOUDIA_ROOT_ACCOUNT_ORG_ID=$(aws organizations list-roots | jq -r '.Roots[0].Id')
# Value would be r-ab12 in this example
aws cloudformation create-stack-instances \
    --stack-set-name cloudia-stack-set \
    --region eu-west-3 \
    --deployment-targets 'OrganizationalUnitIds=["'${CLOUDIA_ROOT_ACCOUNT_ORG_ID}'"]' \
    --operation-preferences "RegionConcurrencyType=PARALLEL,MaxConcurrentCount=5" \
    --regions "eu-west-3"
```

??? note "Output"

    ```raw
    {
        "OperationId": "eaf745f2-b01b-4be9-8e02-b7d39f72416a"
    }
    ```

Once the Stack Instances are created, the IAM Role `cloudia-read-role` is available on all the sub-accounts in the following minutes. Deployment operation takes roughly 1-2 minutes per account.

### Generate data with the Cost and Usage Report (CUR)

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
aws s3api put-bucket-policy --bucket cloudia-ccf-billing-data --policy file://aws_cur_bucket_policy.json
```

#### Create the Cost and Usage Report (CUR) definition

Create the Cost and Usage Report (CUR) definition to generate the data in the Bucket.

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
aws cur put-report-definition --report-definition file://aws_cur_report_definition.json --region us-east-1
```

You can see your report created at the [AWS Cost and Usage Reports (legacy)](https://us-east-1.console.aws.amazon.com/billing/home?region=us-east-1#/reports) page.

Data synchronization can take up to 12 hours to be available in the S3 bucket.
