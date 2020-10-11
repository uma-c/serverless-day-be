# ServerlessDay - Back-end workshop

We build a fully serveless file sharing application with chalice.
This repository contains the back end of the application.

For front-end code and all related statements refer to this repository:

## Prerequisites

- AWS Account
- Three S3 buckets
- A Cognito User Pool
- Two DynamoDB tables


### S3 Bucket

The application will need an S3 bucket for uploads of users' files.
In this case, it will be a private bucket. We then create it in the same way that is used for the front end, and we take note of the bucket name, we will need it for the configuration of the application.

You also need 2 additional buckets that will be used for the CD/CI pipeline, one to hold configurations and one for release bundles.
These two buckets are optional, and should only be created if you decide to implement the pipeline.


### Cognito User Pool

During the Creation Wizard we can customize some behaviors; To make the application work, you need to create a Pool that allows users to register by email and verify the address using a security code managed by Cognito. You also need to create a client app and make a note of its ID.

We choose a name for the User Pool and start the wizard

![Cognito user pool](https://blog.besharp.it/wp-content/uploads/2018/09/SS_19-09-2018_124930.png)


Let's configure the authentication mechanism and profile fields.

![Cognito user pool](https://blog.besharp.it/wp-content/uploads/2018/09/SS_19-09-2018_125142.png)

Finally, you create a client app and make a note of its ID. You don't have to generate any secrets, it won't be used for the use of Cognito through web apps.


![Cognito user pool](https://blog.besharp.it/wp-content/uploads/2018/09/SS_19-09-2018_125238.png)

![Cognito user pool](https://blog.besharp.it/wp-content/uploads/2018/09/SS_19-09-2018_125555.png)


### DynamoDB tables

You will also need a DynamoDB table to hold metadata and share information of your files. We then proceed to create a table by leaving the default values and specifying "owner" with sort key "share_id" as the primary partition key. A global "share_id" index should be added

![Dettagli tabella DynamoDB](https://blog.besharp.it/wp-content/uploads/2018/09/SS_19-09-2018_171125.png)

Another table will be required for the audit log, again we leave all the default settings and indicate as the key before the `action_time`


## Get the code

```bash
git clone https://github.com/uma-c/serverless-day-be.git
```

Or you can download a zip using the button at the top right of the github page.


## Setup

To set the ARN and resource IDs to be used, there is a configuration file in ` ./mydoctransfer-lambda/chalicelib/config.py `

```python
# Cognito
COGNITO_USER_POOL_NAME = 'CHANGE ME'
COGNITO_USER_POOL_ARN = 'CHANGE ME'
COGNITO_USER_POOL_ID = 'CHANGE ME'


# DynamoDB
DYNAMODB_FILE_TABLE_NAME = 'CHANGE ME'
DYNAMODB_AUDIT_TABLE_NAME = 'CHANGE ME'


# S3
S3_UPLOADS_BUCKET_NAME = 'CHANGE ME'
```

Indicate ARN, Name, or ID as required.


## Local mode

The bundle also includes a docker-compose.yml and a Dockerfile that you can use to start the backend locally.

```bash
docker-compose up 
```

If you prefer to avoid docker and install dependencies on your system, you must install python 3.6, pip, and then all dependencies contained in requirements.txt. We strongly recommend that you use a dedicated **VirtualEnv** in case Docker is not a viable option.

```bash
pip3 install -r requirements.txt --user
```


## Deploy

Chalice can deploy and configure Gateway and Lambda APIs; configurations are expressed idiomatically in the application code.

Once the project is ready, you can deploy automatically simply by invoking

```bash
chalice deploy
```

The command must be invoked by an IAM User or Role with permissions to create and manage Lambda Function, Gateway API, and CloudWatch. For the workshop, a user with policy `AdministratorAccess` will do more than fine.


## CI/CD (Optional)

We want to complete the architecture by mentioning automatic pipelines for code deployment.

First, you create a new application on CodeBuild and configure it to use the buildspec.yml file in the root of the repository.

 ![Code build](https://blog.besharp.it/wp-content/uploads/2018/10/SS_09-10-2018_120438.png)

You must pass the following environment and enhance them using the bucket name for configurations and releases.:\

Variables to CodeBuild:\
`CONFIG_S3_BUCKET`\
`RELEASES_S3_BUCKET`

Once CodeBuild is created and configured, just create a Pipeline that sources from CodeCommit and then invokes CodeBuild. you don't need to specify the deployment stage, because for this example we've included the statements in buildspec.yml so that they run after the build process.

Therefore, IAM roles associated with CodeBuild must have permission to upload and delete files from S3, manage Gateway AND Lambda APIs.

By analyzing the buildspec we can see the operations performed for the build and deploy of applications.

```yaml
version: 0.1
phases:
  install:
    commands:
    - rm -rf mydoctransfer-lambda/.chalice/deployed
    - mkdir mydoctransfer-lambda/.chalice/deployed
    - cd mydoctransfer-lambda/.chalice/deployed && aws s3 sync --delete s3://$RELEASES_S3_BUCKET/backend/chalice .
    - cd mydoctransfer-lambda && pip3.6 install -r requirements.txt --user
    - aws s3 cp s3://$CONFIG_S3_BUCKET/backend/$ENV/modsam.py .
    - cd mydoctransfer-lambda/.chalice && aws s3 cp s3://$CONFIG_S3_BUCKET/backend/$ENV/config.json .
    - ./build.sh
    - cd mydoctransfer-lambda/.chalice/deployed && aws s3 sync --delete . s3://$RELEASES_S3_BUCKET/backend/chalice
artifacts:
  type: zip
  files:
    - build.sh
```
