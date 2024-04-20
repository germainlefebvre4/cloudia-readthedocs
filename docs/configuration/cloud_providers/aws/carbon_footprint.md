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
    --template-body file://./cloudia-read-role.yaml \
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
