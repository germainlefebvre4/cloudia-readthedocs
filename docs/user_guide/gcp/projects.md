# GCP Projects

## Principles

Retrieving Cloud Projects requires some kind of rights on the Cloud Provider platform. You need to provide the platform with the credentials to access the Cloud Provider platform in order to retrieve the Cloud Projects.

Each supported Cloud Provider platform has its own way to provide access to the platform. The platform supports the following Cloud Provider platforms.

## Configuration

Requirements:

* Google Cloud Organization ID
* Google Cloud Project Name

Values that will be used in the following configuration:

* Google Cloud Organization ID: `123456789012`
* Google Project Name: `cloudia-project`
* Service Account Name: `cloudia-svc`

Choose a Google Cloud Platform project to store the assets relating to Cloudia.

```bash
gcloud config set project cloudia-project
```

### Grant Service Account access to the Organization Projects

Assign organizations level permission to the service account.

```bash
gcloud organizations add-iam-policy-binding 123456789012 --member="serviceAccount:cloudia-svc@cloudia-project.iam.gserviceaccount.com" --role="roles/resourcemanager.folderViewer"
```
