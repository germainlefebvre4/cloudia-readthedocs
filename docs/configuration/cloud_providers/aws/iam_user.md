# AWS IAM User

## Principles

The IAM User will allow retrieving information from AWS using the API of the Cloud Provider.

## Configuration

Requirements:

* Amazon Web Services Organization ID
* Amazon Web Services Project Name

Values that will be used in the following configuration:

* Amazon Web Services Account Name: `cloudia-project`
* IAM User Name (Service Account): `cloudia-svc`

Choose a Amazon Web Services Platform project to store the assets relating to Cloudia.

```bash
aws configure
```

### Create IAM User and generate credentials

Create a IAM User (that will be used as service account).

```bash
aws iam create-user --user-name cloudia-svc
```

```json
{
    "User": {
        "Path": "/",
        "UserName": "cloudia-svc",
        "UserId": "xxxxxxxxxxxxxxxxxxxx",
        "Arn": "arn:aws:iam::123456789012:user/cloudia-svc",
        "CreateDate": "2023-12-20T21:52:51+00:00"
    }
}
```

Generate the access key ID and secret access key.

```bash
aws iam create-access-key --user-name cloudia-svc
```

```json
{
    "AccessKey": {
        "UserName": "cloudia-svc",
        "AccessKeyId": "xxxxxxxxxxxxxxxxxxxx",
        "Status": "Active",
        "SecretAccessKey": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
        "CreateDate": "2023-12-20T21:55:50+00:00"
    }
}
```

The credentials `AccessKeyId` and `SecretAccessKey` will be used in the API through the `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` environment variable.
