# AWS Multi-Account Demo
Documentation and configuration demonstrating an AWS multi-account setup.  
- [Introduction](#Introduction) - Why implement an aws multi-account strategy.
- [AWS Master Account](#aws-master-account) - Some good practices to setup an AWS Master Account.
- [AWS Organisation](#aws-organisation) - How to setup an AWS organisation using AWS Control Tower.

## Introduction 
It is well documented that is is a good idea to employ a multi-account strategy to logically separate workloads and the environments of those.

The key benefits are:
* Reduce the risk of accidents by having physical separation between different projects.
* Limit the blast radius of security breaches or mistakes
* Better management of AWS Service Quotas and limits
* Consolidated budgets

For full docs see [Benefits of using multiple AWS accounts](https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/benefits-of-using-multiple-aws-accounts.html).

## AWS Master Account
The Master account underpins everything, so it is essential to employ good security practices right from the top.  The approach (or philosophy) to security that this setup uses is:
* *Protect* - Build defence mechanisms around the account to prevent malicious actors entering your account
* *Alert* - Use notifications to be aware when malicious activity is occurring
* *Record* - Log important account activity to provide a historical record.
* *Auditing* - Regularly check that protections are in place and haven't been changed.

Specific examples of this approach are:
1. AWS Root Account security best practices (**Protect**)
1. Configure an IAM user with MFA (**Protect**)
1. Create a password policy for IAM users. (**Protect**)
1. Disable Security Token Service for regions that won't be used. (**Protect**)
1. Root account usage alerts (**Alerts**)
1. Billing alerts (**Alert**)
1. Log management API activity in your AWS Account (**Record**)

AWS initial setup guides you through to ensure a good level of security of the account.  The [AWS Secure Initial Account Setup](https://aws.amazon.com/answers/security/aws-secure-account-setup/), [blog article](https://aws.amazon.com/blogs/security/getting-started-follow-security-best-practices-as-you-configure-your-aws-resources/) and IAM [docs](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html) all provide good information for getting started with this.

### Configure AWS Root Account Security
The AWS Root Account is configured with a strong password, which is stored in a password safe.  Additionally MFA is also configured with a Yubikey device.  After the initial IAM user is setup in the next task the Root account should not need to be used accept in real emergencies.

### Create IAM Admin user
An IAM admin user is setup again with a strong password stored in a password safe.  MFA is also configured with a Yubikey device.  This user is given Administrator permissions and is used to perform the initial AWS Control Tower setup.  Once SSO is implemented this account is rarely used.

### Create a IAM Password Policy
A [password policy](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_passwords_account-policy.html) for IAM users.

Apply password policy:
```
$ aws iam update-account-password-policy --minimum-password-length 12 --require-numbers --require-symbols --require-uppercase-characters --require-lowercase-characters --allow-users-to-change-password --max-password-age 90 --password-reuse-prevention 10 --no-hard-expiry
```

### Disable Security Token Service for unused regions
This is performed in the AWS console using the IAM service.  In Account Settings there is a Security Token Service section where this can be configured.

Currently only the following endpoints are active:
* US East (N. Virginia)
* EU (Ireland)

### Configure logging for management API activity and Root Account Usage Alerts
The [AccessMonitor.yaml](https://github.com/Alex-Burgess/MyAwsOrganisation/blob/master/MasterAccountStacks/AccessMonitor.yaml) template sets up a CloudTrail to watch for management API activity and CloudWatch Alarm based on a metric looking for Root Console activity.

Get the organisation id:
```
aws organizations describe-organization --profile master-admin
```

Create the stack:
```
aws cloudformation create-stack --region us-east-1 --stack-name AccountMonitor \
 --template-body file://AccessMonitor.yaml \
 --capabilities CAPABILITY_NAMED_IAM \
 --parameters \
    ParameterKey=OrgId,ParameterValue=o-123456 \
    ParameterKey=EmailAddress,ParameterValue=john.smith@gmail.com
```

**Note:** An older version of this script used to alert if there were any failed Console or CLI attempts, but this just produced false positives, so deemed not particularly beneficial.  See [Old AccessMonitor.yaml](https://github.com/Alex-Burgess/AwsOrganisation/blob/master/MasterAccountStacks/AccessMonitor.yaml) for reference.

### Configure Billing alerts
The [Budgets.yaml](https://github.com/Alex-Burgess/MyAwsOrganisation/blob/master/MasterAccountStacks/Budgets.yaml) template creates a number of budget alerts.

Create the stack:
```
aws cloudformation create-stack --stack-name Budgets \
 --template-body file://Budgets.yaml \
 --parameters \
    ParameterKey=EmailAddress,ParameterValue=john.smith@gmail.com \
    ParameterKey=PhoneNumber,ParameterValue=+447123567890
```

## AWS Organisation
AWS Control Tower is used to create and administer the multi-account environment. It creates an Organisation, which allows accounts to be grouped into OUs and it sets up SSO to permission users to the accounts.

![alt text](../main/images/Organizations_Architecture.png "Organizations Architecture")

### Guardrails
The *Deny access to AWS based on the requested AWS Region* guardrail is used to restrict access to AWS services to just a few specific regions.  There are a few reasons why one might do this, compliance, data restrictions, etc, but in this scenario it's main benefit is that it reduces the scope of any potential bad actor.  See [restrict regions](https://controltower.aws-management.tools/security/restrict_regions/) for more information.

### AWS SSO
A basic setup is currently employed, with just one user. To see the users and groups go to the AWS SSO console.

To access the AWS SSO login console go to: [https://alexburgess.awsapps.com/start](https://alexburgess.awsapps.com/start)

To configure the CLI with SSO see: [cli-configure-sso.html](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-sso.html)


### Service Control Policies
An SCP is applied to the sandbox ou and accounts for baseline security. Some useful SCP procedures are supplied below.  Assigning SCPs to accounts or OUs is typically easiest in the console.

List polcies:
```
$ aws organizations list-policies --filter SERVICE_CONTROL_POLICY
```

Create New SCP:
1. Define the new policy, e.g. `scp-test-policy.json`
1. Create a new policy with name and description:
    ```
    $ aws organizations create-policy --content file://scp-test-policy.json --description "quick test policy" --name "test" --type SERVICE_CONTROL_POLICY
    ```

Update SCP:
```
aws organizations update-policy --policy-id p-12345678 --content file://sandbox-ou-scp.json
```
