<p>
<center>
<table>
  <tr>
    <td style="text-align: center; vertical-align: middle;"><img src="_docs/logo_aws.jpg"/></td>
    <td style="text-align: center; vertical-align: middle;"><img src="_docs/logo_adv.jpg"/></td>
  </tr> 
<table>
</center>
</p>

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

In order to activate the rotation feature it is necessary to do the following actions before the terraform deployment

### I.1 - Add tag on user

To identify an AWS user as a user with ID rotation, it is necessary to add a tag to this user. This tag must be **IamRotateCredentialEmail**. It must contain the email that will receive the new credentials.

![alt text](_docs/tag.png)

It is possible to configure per user the maximum duration for console access or for command line access

| Name | Description | Required |
|------|-------------|:----:|
| IamRotateCredentials:Email | Email of the user who will receive the new credentials | yes |
| IamRotateCredentials:LoginProfileTimeLimit | Maximum duration for an access with login profile (expressed in days). | no |
| IamRotateCredentials:CliTimeLimit | Maximum duration for an access with AWS CLI (expressed in days). | no |

### I.2 - Register Email/Domain on AWS SES

Once the tags is affixed to the user, the email or email domain must be registered in the AWS SES sevice. Otherwise no mails will be sent from AWS.

### I.2.1 - Register Email

![alt text](_docs/ses.png)

### I.2.2 - Register Domain

![alt text](_docs/ses2.png)

## II - Inputs / Outputs

## Inputs

| Name | Description | Type | Default |
|------|-------------|:----:|:-----:|
| aws\_cli\_time\_limit | Maximum duration for an access with AWS CLI (expressed in days). | number | 60 |
| aws\_login\_profile\_password\_reset\_required | Requires that the console password be changed by the user at the next login. | bool | true |
| aws\_login\_profile\_time\_limit | Maximum duration for an access with login profile (expressed in days). | number | 60 |
| aws\_region | aws region to deploy (only aws region with AWS SES service deployed) | string | n/a |
| aws\_ses\_email\_from | email used to send emails to users when their credentials change. | string | n/a |
| cloudwatch\_log\_retention | The cloudwatch log retention ( default 7 days ). | number | 7 |
| credentials\_sended\_by | The sender of renewal credentials emails | string | "ops team" |
| function\_timeout | The amount of time your Lambda Functions has to run in seconds. | number | 300 |
| scan\_alarm\_clock | The time between two scan to search for expired certificates ( in minutes default 1440 = 1 days) | number | 1440 |
| tags | The tags of all resources created | map | {} |

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
  tag = {
    Owner = "Acme"
    Department = "ops"
  }
}
````
