# AWS Billing

## Principles

Retrieving Cloud Projects Billing requires actions on specified Cloud Projects. You need to provide the platform with the credentials to access the Cloud Provider dedicated to the Billing scope in order to retrieve the billing data.

## Configuration

You have set up the Cloud Provider Access explained in page [AWS Accounts](projects.md) section Amazon Web Services.

### Grant Service Account access to Cost Explorer

On the **Root Account**, grant the `CostAndUsageReport` policy to the IAM User (Service Account) `cloudia-svc`.

* IAM User name: `cloudia-svc`
* Policy name: `allowCostAndUsageReportReadOnly`
* Policy Content:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ce:GetCostAndUsage"
            ],
            "Resource": "*"
        }
    ]
}
```
