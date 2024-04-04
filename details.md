# Driftctl Pipeline for Bitbucket

## Introduction

\```
# 1. This pipeline will work with Bit-Bucket
# 2. Change the command at 19 with your bucket name if You are using AWS S3 !!
# 3. You need to provide 3 things after completing  2nd instruction AWS_ACCESS_KEY_ID , AWS_SECRET_ACCESS_KEY ,  SLACK_WEBHOOK_URL
\```

\```
This pipeline is designed to work with Bitbucket, and it includes instructions for using it with AWS S3 for storing Terraform state files. You need to provide three things: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, and `SLACK_WEBHOOK_URL`.
\```

## Pipeline Configuration

\```yaml
image: snyk/driftctl

pipelines:
  branches:
    driftctl:
\```

\```
The pipeline is configured to run for all branches in the repository. It uses the `snyk/driftctl` Docker image, which is a tool for detecting drift (unmanaged resources) in your infrastructure compared to your Terraform state files.
\```

## Snyk Driftctl Scan Step

\```yaml
    - step:
        name: Snyk Driftctl Scan
        script:
        - echo ${BITBUCKET_STEP_UUID//[\{\}]} > artifact_uuid.txt
        - export DCTL_TF_PROVIDER_VERSION=5.33.0
        - export AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
        - export AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
        - export AWS_REGION=eu-central-1
        - export DCTL_OUTPUT=json://results.json
        - "set +e\ndriftctl scan --from tfstate+s3://<Your_company_name>-terraform-state/terraform.tfstate --from tfstate+s3://<Your_company_name>-terraform-access/terraform.tfstate # This command should change \nSCAN_EXIT_CODE=$?\nset -e\necho $SCAN_EXIT_CODE\nif [ $SCAN_EXIT_CODE -eq 0 ] || [ $SCAN_EXIT_CODE -eq 1 ]; then\n  echo \"Scan completed successfully with exit code: $SCAN_EXIT_CODE\"\nelse\n  echo \"Scan failed with exit code: $SCAN_EXIT_CODE\"\n  exit $SCAN_EXIT_CODE\nfi\n"
        artifacts:
        - results.json
        - artifact_uuid.txt
\```

\```
This step performs the actual Driftctl scan. It saves the unique identifier (UUID) of the current step to a file called `artifact_uuid.txt`, sets the Terraform provider version, exports the provided AWS credentials and region, configures the output of the scan to be stored in a JSON file called `results.json`, and runs the `driftctl scan` command to compare the resources in your AWS account with the resources defined in your Terraform state files. It also checks the exit code of the scan command and exits with the same code if the scan failed. Finally, it saves the `results.json` file and the `artifact_uuid.txt` file as artifacts.
\```

## Send to Slack Step

\```yaml
    - step:
        name: Send to Slack
        image: ubuntu:latest
        script:
        - |
          export ARTIFACT_UUID=$(cat artifact_uuid.txt)
          apt update && apt install jq curl -y
          export SCAN_DATE=$(date +"%d %B %H:%M %Z")
          export DOWNLOAD_LINK="https://bitbucket.org/$BITBUCKET_REPO_OWNER/$BITBUCKET_REPO_SLUG/pipelines/results/$BITBUCKET_BUILD_NUMBER/steps/%7B$ARTIFACT_UUID%7D/artifacts"
          # Check if results.json exists
          if [ -f "results.json" ]; then
            export TOTAL_RESOURCES=$(jq '.summary.total_resources' results.json)
            export MANAGED_RESOURCES=$(jq '.summary.total_managed' results.json)
            export UNMANAGED_RESOURCES=$(jq '.summary.total_unmanaged' results.json)

            # Check if there are any unmanaged resources
            if jq -e '.unmanaged | length > 0' results.json > /dev/null; then
              UNMANAGED_DETAILS=$(
                jq -r '.unmanaged | group_by(.type) | map({type: .[0].type, resources: [.[].id]}) | map("\n\(.type)\n\(.resources | map("  - \(.)")|join("\n"))")[]' results.json
              )

              # Build Slack message
              SLACK_MESSAGE='{
              "attachments": [
              {
                "blocks": [
                  {
                    "type": "header",
                    "text": {
                      "type": "plain_text",
                      "text": "<Your_company_name> - 331454485377\nDriftctl Scan Results: '$SCAN_DATE'"
                  }
                  },
                  {
                    "type": "divider"
                  },
                  {
                    "type": "section",
                    "text": {
                      "type": "mrkdwn",
                      "text": "*Total Resources:* '$TOTAL_RESOURCES'\n*Managed Resources:* '$MANAGED_RESOURCES'\n*Unmanaged Resources:* '$UNMANAGED_RESOURCES'\n\n*Unmanaged Details:*\n```'$UNMANAGED_DETAILS'```"
                    }
                  },
                  {
                    "type": "actions",
                    "elements": [
                      {
                        "type": "button",
                        "text": {
                          "type": "plain_text",
                          "text": "Download Report",
                          "emoji": true
                        },
                        "url": "'$DOWNLOAD_LINK'",
                        "style": "primary"
                      }
                    ]
                  }
                ],
                "color": "3498DB"
                }
              ]
              }'
              curl -X POST -H 'Content-type: application/json' --data "$SLACK_MESSAGE" "$SLACK_WEBHOOK_URL"
            else
              echo "No unmanaged resources found. Skipping Slack Notification."
            fi
          else
            echo "results.json not found. Skipping Slack Notification."
          fi
\```

\```
This step is responsible for sending a notification to a Slack channel if there are any unmanaged resources found during the scan. It retrieves the UUID of the previous step from the `artifact_uuid.txt` file, checks if the `results.json` file exists, extracts the total number of resources, managed resources, and unmanaged resources from the JSON file, and constructs a Slack message with details about the unmanaged resources if there are any. It then sends the Slack message to the provided `SLACK_WEBHOOK_URL` using `curl`. If there are no unmanaged resources or if the `results.json` file is not found, it skips sending the Slack notification.
\```

\```
The Slack message includes the following information: header with the company name and scan date, total number of resources, managed resources, and unmanaged resources, details of unmanaged resources (type and IDs) in a code block, and a button to download the report from the Bitbucket pipeline artifacts.
\```
