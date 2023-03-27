- [3Scale installation overview](#3scale-installation-overview)
  - [Creating a new OpenShift project](#creating-a-new-openshift-project)
  - [Installing and configuring the 3scale operator](#installing-and-configuring-the-3scale-operator)
    - [OperatorGroup](#operatorgroup)
    - [Subscription](#subscription)
  - [Storage](#storage)
    - [AWS S3 file storage installation](#aws-s3-file-storage-installation)
    - [Amazon S3 secret](#amazon-s3-secret)
    - [AWS EFS file storage installation](#aws-efs-file-storage-installation)
  - [Creating 3scale APIManager instance](#creating-3scale-apimanager-instance)
  - [Admin portal credential](#admin-portal-credential)
  - [Products and backends for 3scale APIs](#products-and-backends-for-3scale-apis)
    - [Create backend](#create-backend)
    - [Create product](#create-product)
    - [Create routes](#create-routes)
  - [Creating 3scale audience account](#creating-3scale-audience-account)
  - [Creating 3scale application plans](#creating-3scale-application-plans)
  - [Creating 3scale product application](#creating-3scale-product-application)
  - [Publish 3Scale configuration](#publish-3scale-configuration)
    - [Staging configuration](#staging-configuration)
    - [Production configuration](#production-configuration)

# 3Scale installation overview

Before installation, please replace the domain `<WILDCARD_DOMAIN>` in `resources` folder.

To get wildcard domain you can use the following command

```shell
oc get ingresses.config cluster -o jsonpath='{.spec.domain}'
```

In the instructions, we will use the `apps.sssc-cl01.appeng.rhecoeng.com` domain

## Creating a new OpenShift project

```shell
oc new-project 3scale-project
```

## Installing and configuring the 3scale operator

You can use the 3Scale [documentation](https://access.redhat.com/documentation/en-us/red_hat_3scale_api_management/2.13/html/installing_3scale/installing-threescale-operator-on-openshift) to install operator or use the following configuration.

### OperatorGroup

```yaml
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  generateName: 3scale-project-
  namespace: 3scale-project
```

### Subscription

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: 3scale-operator
  namespace: 3scale-project
spec:
  channel: threescale-2.13
  installPlanApproval: Automatic
  name: 3scale-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  startingCSV: 3scale-operator.v0.10.1-0.1675914645.p
```

## Storage

Storage can be configured in two ways - NFS server and AWS S3 Bucket.

In our case we will use S3.

### AWS S3 file storage installation

- Create an Identity and Access Management (IAM) policy with the following permissions.
- Link the created policy to the required account in AWS

Replace `targetBucketName`, `<accountName>` and `<userName>` with the appropriate value.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": "s3:*",
      "Effect": "Allow",
      "Resource": "arn:aws:s3:::<targetBucketName>",
      "Principal": {
        "AWS": [
          "arn:aws:iam::<accountName>:user/<userName>"
        ]
      }
    }
  ]
}
```

### Amazon S3 secret

Replace the placeholders in the secret with the appropriate values

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: 3scale-aws-s3
  namespace: 3scale-project
stringData:
  AWS_ACCESS_KEY_ID: <>
  AWS_SECRET_ACCESS_KEY: <>
  AWS_BUCKET: <>
  AWS_REGION: <>
type: Opaque
```

### AWS EFS file storage installation

For NFS installation and configuration please use the following links

[3Scale environment requirements](https://access.redhat.com/documentation/en-us/red_hat_3scale_api_management/2.13/html/installing_3scale/install-threescale-on-openshift-guide#environment_requirements)

[Amazon Elastic File System](https://docs.aws.amazon.com/efs/latest/ug/getting-started.html)

## Creating 3scale APIManager instance

```shell
oc create -f resources/api-manager-s3.yaml
```

## Admin portal credential

```shell
oc get secret system-seed -o json | jq -r .data.ADMIN_USER | base64 -d
oc get secret system-seed -o json | jq -r .data.ADMIN_PASSWORD | base64 -d
```

## Products and backends for 3scale APIs

### Create backend

```shell
oc create -f resources/backend.yaml
```

### Create product

Before create Product you must replace `<SECRET_PLACEHOLDER>` with the appropriate value.

This value defined in `THREESCALE_ACCOUNT_SECRET` [environment variable](https://github.com/fabric8-analytics/fabric8-analytics-server/blob/master/openshift/template.yaml#L115)

```shell
oc create -f resources/product.yaml
```

### Create routes

```shell
oc create -f resources/crda-api-production-route.yaml
oc create -f resources/crda-api-staging-route.yaml
```

## Creating 3scale audience account

- Navigate to Audience > Accounts > Listing.
- Click on "Create" button.
- Fill the form with username, email, password and use `External` as Organization name.
- Click the Create button.

## Creating 3scale application plans

We need to create two application plans: `free-tier` and `premium-tier`.

- Navigate to [Product_Name] > Applications > Application Plans.
- Click Create Application Plan.
- On the Create Application Plan page, enter a name and a system name for your new plan. A system name must be unique in your 3scale installation.
- Click Create Application Plan.
- Choose `free-tier` as Default plan.
- Click on three-dots context menu and click "Publish".

## Creating 3scale product application

- Navigate to Products > [Product_Name] > Applications -> Listing.
- Click Create application.
- Select the `External` account for the application.
- Select the `free-tier` application plan.
- Specify `common` as an application name.
- In the Description field enter `Default application for all users`.
- Click Create application.
- Copy the `User Key` for the created application.

## Publish 3Scale configuration

### Staging configuration

- Navigate to Products > [Product_Name] > Integration -> Configuration.
- Click on `Promote v.X to Staging APIcast`

Run the following command to check deployed configuration

Replace `<USER_KEY>` with the value of the `User Key` from the `common` application.

```shell
curl --location 'http://api-staging.apps.sssc-cl01.appeng.rhecoeng.com/api/v2/component-analyses/maven/log4j:log4j/1.2.7?user_key=<USER_KEY>'
```

if you received a response from the server, you must publish the configuration to production.

### Production configuration

- Navigate to Products > [Product_Name] > Integration -> Configuration.
- Click on `Promote v.X to Production APIcast`

Run the following command to check deployed configuration

Replace `<USER_KEY>` with the value of the `User Key` from the `common` application.

```shell
curl --location 'http://api-staging.apps.sssc-cl01.appeng.rhecoeng.com/api/v2/component-analyses/maven/log4j:log4j/1.2.7?user_key=<USER_KEY>'
```
