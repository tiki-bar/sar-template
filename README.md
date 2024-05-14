# AWS SAR Template
Publish and share [Organization](https://aws.amazon.com/organizations/) wide [Cloudformation](https://aws.amazon.com/cloudformation/) templates using the [Serverless Application Repository (SAR)](https://aws.amazon.com/serverless/serverlessrepo/). 

## Setup
Assumes you already have AWS [Organization](https://aws.amazon.com/organizations/) with a dedicated Identity account with a [GitHub OIDC](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services) provider configured and a dedicated Artifacts account for your deployable assets.

1) In your Identity account create (if it doesn't already exist) a role (e.g. `GitHubOidc`) that can assume roles in your other accounts.

    #### Trusted entities
    ```json
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Principal": {
            "Federated": "arn:aws:iam::{IDENTITY_ACCOUNT_ID}:oidc-provider/token.actions.githubusercontent.com"
          },
          "Action": "sts:AssumeRoleWithWebIdentity",
          "Condition": {
            "StringEquals": {
              "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
            },
            "StringLike": {
              "token.actions.githubusercontent.com:sub": [
                "repo:{REPOSITORY}"
              ]
            }
          }
        }
      ]
    }
    ```
    #### Permissions
    ```json
    {
      "Version": "2012-10-17",
      "Statement": [
          {
              "Effect": "Allow",
              "Action": ["sts:TagSession", "sts:AssumeRole"],
              "Resource": [
                  "arn:aws:iam::{ARTIFACTS_ACCOUNT_ID}:role/GitHubSarPublish"
              ]
          }
      ]
    }
    ```
2) In your Artifacts account create a role (e.g. `GitHubSarPublish`) for publishing to SAR.

    #### Trusted entities
    ```json
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Principal": {
                  "AWS": "arn:aws:sts::{IDENTITY_ACCOUNT_ID}:assumed-role/OrganizationFormationBuildAccessRole/GitHubActions"
              },
              "Action": ["sts:TagSession", "sts:AssumeRole"]
          }
        ]
    }
    ```
    #### Permissions
    ```json
    {
        "Version": "2012-10-17",
        "Statement": [
          {
            "Effect": "Allow",
            "Action": [
                "serverlessrepo:ListApplications",
                "serverlessrepo:CreateApplication",
                "serverlessrepo:SearchApplications"
             ],
            "Resource": "*"
          },
          {
            "Effect": "Allow",
            "Action": "serverlessrepo:*",
             "Resource": "arn:aws:serverlessrepo:*:{ARTIFACTS_ACCOUNT_ID}:applications/*"
          }
        ]
    }
    ```

3) Create an [S3](https://aws.amazon.com/s3/) Bucket to host your [Cloudformation](https://aws.amazon.com/cloudformation/) templates. (e.g `mytiki-artifacts-sar`)
4) Update the Bucket Policy to provide CRUD access for the `GitHubSarPublish` role and read access for `serverlessrepo.amazonaws.com`
```json
{
  "Version": "2012-10-17",
  "Statement": [
  {
    "Effect": "Allow",
    "Principal": {
      "Service":  "serverlessrepo.amazonaws.com"
    },
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::<your-bucket-name>/*",
    "Condition" : {
      "StringEquals": {
        "aws:SourceAccount": "{ARTIFACTS_ACCOUNT_ID}"
      }
    }
  },
  {
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::{ARTIFACTS_ACCOUNT_ID}:role/GitHubSarPublish"
    },
    "Action": [
      "s3:ListBucket",
      "s3:GetObject",
      "s3:GetObjectVersion",
      "s3:PutObject",
      "s3:DeleteObject"
    ],
    "Resource": [
      "arn:aws:s3:::mytiki-artifacts-sar/*",
      "arn:aws:s3:::mytiki-artifacts-sar"
      ]
    }
  ]
}
```

## How to Use
1) Copy the [template.yml](template.yml), [samconfig.toml](samconfig.toml), and [.github/workflows/publish.yml](.github/workflows/publish.yml) files to your project.
2) Set the `s3_bucket` value in [samconfig.toml](samconfig.toml) to your artifacts bucket
3) Add your cloudformation resources to [template.yml](template.yml)
4) Update the roles to assume, GitHub secrets, and environment variables in [publish.yml](.github/workflows/publish.yml)
5) Push to main. A new application will be available in all org accounts under `Serverless Application Repository > Available Applications > Private applications`.
