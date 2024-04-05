# Introduction 
## Driftctl is a tool that helps you detect infrastructure drift between your actual cloud resources and your Terraform configuration. Infrastructure drift occurs when the state of your cloud resources differs from what is defined in your Terraform code, leading to potential security risks and inconsistencies.

### This code is a pipeline script that performs the following tasks:

+ 1. It scans your AWS resources using the Driftctl tool and compares them with the Terraform state files stored in S3 buckets.
+ 2. The scan results are saved in a JSON file, which includes information about managed and unmanaged resources.
+ 3. If there are any unmanaged resources (resources that exist in the cloud but are not managed by Terraform), the script generates a detailed report.
+ 4. The report is then sent to a Slack channel using a webhook URL, providing information about the total resources, managed resources, unmanaged resources, and a list of unmanaged resource details.
+ 5. The report also includes a button that allows you to download the full scan results JSON file from the pipeline artifacts.

+ ### 1. This pipeline will work with Bit-Bucket
+ ### 2. Change the command at with your bucket names and replace placeholders below
+ ### 3. Configure environment variables in your Bitbucket project settings
+ ### 4. Update the AWS region as needed

This pipeline is designed to work with Bitbucket, and it includes instructions for using it with AWS S3 for storing Terraform state files. Before using this pipeline, make sure to complete the following prerequisites:

## Prerequisites


1. Create two Amazon S3 buckets named '<Your_company_name>-terraform-state' and '<Your_company_name>-terraform-access'. Make sure that both buckets exist and have the necessary permissions for the pipeline to read and write files.

2. Set up the required environment variables in your Bitbucket project settings under the "Pipeline Settings" tab. Add the following variables and their corresponding values:
   - `AWS_ACCESS_KEY_ID`: Your AWS Access Key ID.
   - `AWS_SECRET_ACCESS_KEY`: Your AWS Secret Access Key.
   - `SLACK_WEBHOOK_URL`: The URL of your Slack webhook. To get this, create an incoming webhook in your Slack workspace and copy its URL.
3. Update the value of the `export AWS_REGION` variable to match the desired AWS region where your infrastructure resides. For instance:

```bash
export AWS_REGION=us-west-2
