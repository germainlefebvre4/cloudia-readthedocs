# AWS Accounts

## Principles

Retrieving Cloud Projects requires some kind of rights on the Cloud Provider platform. You need to provide the platform with the credentials to access the Cloud Provider platform in order to retrieve the Cloud Projects.

Each supported Cloud Provider platform has its own way to provide access to the platform. The platform supports the following Cloud Provider platforms.

## Configuration

Requirements:

* Amazon Web Services Organization ID
* Amazon Web Services Project Name

Values that will be used in the following configuration:

* Amazon Web Services Organization ID: `123456789012`
* Amazon Web Services Account Name: `cloudia-project`
* IAM User Name (Service Account): `cloudia-svc`

Choose a Amazon Web Services Platform project to store the assets relating to Cloudia.

```bash
aws configure
```

### Grant User access to the Organization Accounts

Attach IAM policy `AWSOrganizationsReadOnlyAccess` to the IAM User.

```bash
aws iam attach-user-policy --user-name cloudia-svc --policy-arn arn:aws:iam::aws:policy/AWSOrganizationsReadOnlyAccess
```
