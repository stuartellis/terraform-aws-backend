# terraform-aws-backend

Tooling to deploy an [AWS backend for Terraform state](https://www.terraform.io/language/settings/backends/s3), using [Ansible](https://www.ansible.com/) and CloudFormation.

This sets up the AWS resources that Terraform needs, in a way that does not depend on Terraform.

The CloudFormation templates deploy a set of replicated S3 buckets and a DynamoDB table. Each bucket has [object versioning](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html) enabled, so that previous versions of the state files can be restored. All storage is encrypted with KMS customer-managed keys.

To increase the safety of the Terraform state, the policy on the main S3 bucket prevents objects from being deleted.

Ansible provides a convenient way to deploy sets of CloudFormation templates together. You may use the CloudFormation templates without Ansible.

## Design

This assumes that you have at least two AWS accounts:

- A *managing* AWS account that will contain the Terraform backend
- One or more *managed* AWS accounts that will contain resources that will be managed by Terraform

This is good practice. AWS accounts are free, because you are only charged for the resources that you use.

If you wish, you can use Terraform to control resources in the managing AWS account. The aim of this tooling is to set up just the AWS resources that Terraform itself needs.

The deployed resources include an IAM user account. This account can assume the roles to access the Terraform storage, and execute Terraform on each AWS account. Your automation can then use this account to run Terraform on AWS.

These templates can be adapted to support larger numbers of AWS accounts.

The *org_prefix* should be a short string that uniquely identifies the current organization. S3 bucket names must be globally unique across all AWS customers, so we must prefix the name of each bucket with a string that is unique to our accounts. These templates also use the *org_prefix* to namespace AWS tags.

## Setting Up Ansible

Ansible requires Python 3. You may run Ansible on Linux, macOS or WSL.

Run these commands to install Ansible:

    pip3 --user pipx
    pipx install ansible-core
    pipx inject ansible-core boto3

Install the extra Ansible collection for AWS:

    ansible-galaxy collection install amazon.aws

## Usage

### Requirements

First decide an *org_prefix*, a short string that uniquely identifies your organization. This is used to ensure consistent and unique naming.

Choose appropriate regions for your resources. The examples use eu-west-2 as the AWS region for the main bucket, with replica buckets in the eu-west-1 and eu-central-1 regions.

### Deploy IAM Roles for Running Terraform

Create the IAM role that Terraform will use to run on each AWS account. To run the Ansible playbook for Terraform execution role on an AWS account:

    ansible-playbook --connection=local ./ansible/deploy-tf-exec-role.yml --extra-vars "managing_account_id=MANAGING-AWS-ACCOUNT-ID org_prefix=YOUR-ORG-PREFIX stack_prefix=tf-exec"

### Deploy Terraform Storage to the Managing Account

Run the Ansible playbook for Terraform storage:

    ansible-playbook --connection=local ./ansible/deploy-tf-storage-playbook.yml --extra-vars "org_prefix=YOUR-ORG-PREFIX stack_prefix=tf-state primary_region=eu-west-2 replica_region_001=eu-west-1 replica_region_002=eu-central-1"

Get the ARNs of the resources that were generated by this playbook.

Run the Ansible playbook to deploy access to the Terraform backend storage:

    ansible-playbook --connection=local ./ansible/deploy-tf-backend-access.yml --extra-vars "org_prefix=YOUR-ORG-PREFIX stack_prefix=tf-access-svc managing_account_id=MANAGING-AWS-ACCOUNT-ID kms_key_arn=PRIMARY_KMS_KEY_ARN s3_bucket_arn=arn:aws:s3:::YOUR-ORG-PREFIX-tf-state-primary-MANAGING-AWS-ACCOUNT-ID-eu-west-2 ddb_table_arn=arn:aws:dynamodb:eu-west-2:MANAGING-AWS-ACCOUNT-ID:table/YOUR-ORG-PREFIX-tf-state-lock-eu-west-2 same_account_exec_role=arn:aws:iam::MANAGING-AWS-ACCOUNT-ID:role/YOUR-ORG-PREFIX-tf-exec-role account_0001_exec_role=arn:aws:iam::A-MANAGED-AWS-ACCOUNT-ID:role/YOUR-ORG-PREFIX-tf-exec-role"

### Deploy IAM Identities for Accessing Terraform to the Managing Account

Run the playbook to deploy IAM users and groups:

    ansible-playbook --connection=local ./ansible/deploy-tf-identities.yml --extra-vars "org_prefix=YOUR-ORG-PREFIX stack_prefix=tf-access-svc managing_account_id=MANAGING-AWS-ACCOUNT-ID backend_access_role=arn:aws:iam::MANAGING-AWS-ACCOUNT-ID:role/YOUR-ORG-PREFIX-tf-backend-access-role
    same_account_exec_role=arn:aws:iam::MANAGING-AWS-ACCOUNT-ID:role/YOUR-ORG-PREFIX-tf-exec-role account_0001_exec_role=arn:aws:iam::A-MANAGED-AWS-ACCOUNT-ID:role/YOUR-ORG-PREFIX-tf-exec-role"

Once this playbook has completed, you will have a new IAM user account. This account can assume the roles to access the Terraform storage, and execute Terraform on each AWS account.

Create an AWS access key for the user, and configure your CI system to use it. Your CI system can then deploy to the AWS accounts with Terraform.

## Resources

- [Terraform documentation for state storage with AWS](https://www.terraform.io/language/settings/backends/s3)
- [Terragrunt requirements for AWS](https://terragrunt.gruntwork.io/docs/features/aws-auth/)
- [CloudFormation for a Terraform backend, by Tibor Hercz](https://github.com/tiborhercz/tf-state-backend-s3-cloudformation) - An existing implementation, using just CloudFormation
- [Managing Terraform Remote State with CloudFormation, by Chris Kent](https://thirstydeveloper.io/tf-skeleton/2021/02/25/part-6-protecting-state.html) - Part of a series on setting up Terraform
