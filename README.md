<table>
  <tr>
    <td style="text-align: center; vertical-align: middle;"><img src="_docs/logo_aws.jpg"/></td>
    <td style="text-align: center; vertical-align: middle;"><img src="_docs/logo_adv.jpg"/></td>
  </tr> 
<table>

# AWS IAM rotate credential

This terraform module aims to create a lambda function that refreshes the IAM credentials (login profile / access keys) as they become obsolete

![alt text](_docs/diagram.png)

## I - Infrastructure components

This module create:

- 2 Lambda functions : **iam-rotate-credentials-update-iam-credentials-for-user**, **iam-rotate-credentials-find-users-to-refresh**

- 2 IAM roles for the lambda function :**iam-rotate-credentials-update-iam-credentials-for-user-role**, **iam-rotate-credentials-find-users-to-refresh-role**

- 2 IAM policies for the iam role :**iam-rotate-credentials-update-iam-credentials-for-user-policy**, **iam-rotate-credentials-find-users-to-refresh-policy**

- 2 Cloudwatch log groups for the logs : **/aws/lambda/iam-rotate-credentials**

- 1 SNS topics for result of lambda function execution : **iam-rotate-credentials-result**

- 2 SQS queues: **iam-rotate-credentials-update-iam-credentials-request**, **iam-rotate-credentials-update-iam-credentials-request-dead-letter**

### I.1 - Lambda environment variables

#### I.1.1 - Lambda : iam-rotate-credentials-find-users-to-refresh

![alt text](_docs/lambda-1.png)

| Name | Description | type | Required |
|------|-------------|:----:|:----:|
| AWS_CLI_TIME_LIMIT | Maximum duration for an access with AWS CLI (expressed in days / default 90 ). | integer | yes |
| AWS_LOGIN_PROFILE_TIME_LIMIT | Maximum duration for an access with login profile (expressed in days / default 90 ). | integer |  no |
| AWS_SNS_RESULT_ARN | The SNS result ARN of topic for result IAM rotate Credential lambdas execution | string | yes |
| AWS_SQS_REQUEST_URL | The ARN of SQS request IAM users credentials | string | yes |

#### I.1.2 - Lambda : iam-rotate-credentials-update-iam-credentials-for-user

![alt text](_docs/lambda-2.png)

| Name | Description | type |  Required |
|------|-------------|:----:|:----:|
| AWS_ACCOUNT_NAME | Name of Aws Account ( use in email sender to user where credentials are obsoletes ) | string | no |
| AWS_LOGIN_PROFILE_PASSWORD_RESET_REQUIRED | Requires that the console password be changed by the user at the next login. | boolean |  yes |
| AWS_SES_EMAIL_FROM | The SNS result ARN of topic for result IAM rotate Credential lambdas execution | string | yes |
| AWS_SNS_RESULT_ARN | The SNS result ARN of topic for result IAM rotate Credential lambdas execution | string | yes |
| CREDENTIALS_SENDED_BY | The SNS result ARN of topic for result IAM rotate Credential lambdas execution | string | yes |

### I.2 - Add tag on user

To identify an AWS user as a user with ID rotation, it is necessary to add a tag to this user. This tag must be **IamRotateCredentials:Email**. It must contain the email that will receive the new credentials.

![alt text](_docs/tag.png)

It is possible to configure per user the maximum duration for console access or for command line access

| Name | Description | type |  Required |
|------|-------------|:----:|:----:|
| IamRotateCredentials:Email | Email of the user who will receive the new credentials | string | yes |
| IamRotateCredentials:LoginProfileTimeLimit | Maximum duration for an access with login profile (expressed in days). | integer | no |
| IamRotateCredentials:LoginProfilePasswordResetRequired | Requires that the console password be changed by the user at the next login.| boolean | no |
| IamRotateCredentials:CliTimeLimit | Maximum duration for an access with AWS CLI (expressed in days). | integer | no |

### I.3 - Register Email/Domain on AWS SES

Once the tags is affixed to the user, the email or email domain must be registered in the AWS SES sevice. Otherwise no mails will be sent from AWS.

#### I.3.1 - Register Email

![alt text](_docs/ses.png)

#### I.3.2 - Register Domain

![alt text](_docs/ses2.png)

### I.4 - Force refresh credentials for one user

For force a credential refresh for one user, you can push message in SQS queue. The message must be like this

```json
{
  "user_name": "<iam user_name>",
  "force": "true"
}
```

## II - Inputs / Outputs

## Inputs

| Name | Description | Type | Default |
|------|-------------|:----:|:-----:|
| aws\_account\_name | Name of Aws Account ( use in email sender to user where credentials are obsoletes ) | string | "<your aws acccount name>" |
| aws\_cli\_time\_limit | Maximum duration for an access with AWS CLI (expressed in days). | number | 90 |
| aws\_login\_profile\_password\_reset\_required | Requires that the console password be changed by the user at the next login. | bool | true |
| aws\_login\_profile\_time\_limit | Maximum duration for an access with login profile (expressed in days). | number | 90 |
| aws\_region | aws region to deploy (only aws region with AWS SES service deployed) | string | n/a |
| aws\_ses\_email\_from | email used to send emails to users when their credentials change. | string | n/a |
| cloudwatch\_log\_retention | The cloudwatch log retention ( default 7 days ). | number | 7 |
| credentials\_sended\_by | The sender of renewal credentials emails | string | "<your ops teams>" |
| function\_timeout | The amount of time your Lambda Functions has to run in seconds. | number | 300 |
| kms\_ciphertext | Data to be encrypted | string | "" |
| scan\_alarm\_clock | The time between two scan to search for expired certificates ( in minutes default 1440 = 1 days) | number | 1440 |
| tags | The tags of all resources created | map | {} |

## Outputs

| Name | Description |
|------|-------------|
| lambda\_find\_users\_to\_refresh\_arn | The Lambda ARN of Find users to update IAM credentials lambda |
| lambda\_update\_iam\_credentials\_for\_user\_arn | The Lambda ARN of Update IAM credentials lambda |
| sns\_iam\_rotate\_credentials\_result\_arn | The SNS result ARN of topic for result IAM rotate Credential lambdas execution |
| sqs\_update\_iam\_credentials\_for\_user\_arn | The ARN of SQS request IAM users credentials |
| sqs\_update\_iam\_credentials\_for\_user\_dead\_letter\_arn | The ARN of SQS request IAM users credentials ( dead letter ) |
| sqs\_update\_iam\_credentials\_for\_user\_dead\_letter\_id | The URL of SQS request IAM users credentials ( dead letter ) |
| sqs\_update\_iam\_credentials\_for\_user\_id | The URL of SQS request IAM users credentials |

## III - Usage

````shell
module "iam_rotate_credentials"
{
  source = "git::https://github.com/AdventielFr/terraform-aws-iam-rotate-credentials.git?ref=1.0.0"
  
  aws_region                                = "eu-west-1"
  cloudwatch_log_retention                  = 10
  aws_cli_time_limit                        = 20
  aws_login_profile_time_limit              = 20
  aws_login_profile_password_reset_required = true
  aws_ses_email_from                        = "no-reply@nobody.com"
  credentials_sended_by                     = "ops team"
  tags = {
    Owner = "Acme"
    Department = "ops"
  }
}
````
