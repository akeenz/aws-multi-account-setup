
# Deploy CloudFormation GitHub Action Workflow

## Overview

This GitHub Actions workflow automates the deployment of AWS CloudFormation stacks for two separate AWS accounts (Account 1 and Account 2). It monitors the `main` branch for changes in specific directories (`account-1/` and `account-2/`) and triggers deployments only when changes are detected in those directories.

## Workflow Breakdown

### Triggers

The workflow is triggered by a `push` event to the `main` branch when changes are detected in the `account-1/` or `account-2/` directories.

\`\`\`yaml
on:
  push:
    branches:
      - main
    paths:
      - account-1/**
      - account-2/**
\`\`\`

### Jobs

The workflow consists of three jobs:

1. **determine-changes**
2. **deploy-account-1**
3. **deploy-account-2**

#### 1. Determine Changes

This job checks if there are changes in the `account-1/` or `account-2/` directories.

- **Checkout code:** Retrieves the latest code from the repository.
- **Set up Environment:** Determines if there are changes in the monitored directories and sets corresponding outputs.

\`\`\`yaml
jobs:
  determine-changes:
    runs-on: ubuntu-latest
    outputs:
      account_1_changed: \${{ steps.set-env.outputs.account_1_changed }}
      account_2_changed: \${{ steps.set-env.outputs.account_2_changed }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v2
        with:
          fetch-depth: 0  # Fetch all history for accurate git diff

      - name: Set up Environment
        id: set-env
        run: |
          echo "Checking for modified files..."
          if [ "\${{ github.event.before }}" = "0000000000000000000000000000000000000000" ]; then
            echo "account_1_changed=true" >> $GITHUB_ENV
            echo "account_2_changed=true" >> $GITHUB_ENV
            echo "::set-output name=account_1_changed::true"
            echo "::set-output name=account_2_changed::true"
          else
            git diff --name-only \${{ github.event.before }} \${{ github.sha }} > files_changed.txt
            if grep -q '^account-1/' files_changed.txt; then
              echo "account_1_changed=true" >> $GITHUB_ENV
              echo "::set-output name=account_1_changed::true"
            else
              echo "account_1_changed=false" >> $GITHUB_ENV
              echo "::set-output name=account_1_changed::false"
            fi
            if grep -q '^account-2/' files_changed.txt; then
              echo "account_2_changed=true" >> $GITHUB_ENV
              echo "::set-output name=account_2_changed::true"
            else
              echo "account_2_changed=false" >> $GITHUB_ENV
              echo "::set-output name=account_2_changed::false"
            fi
          fi
        shell: bash
\`\`\`

#### 2. Deploy to Account 1

This job runs only if changes are detected in the `account-1/` directory.

- **Checkout code:** Retrieves the latest code from the repository.
- **Set up AWS CLI for Account 1:** Configures AWS credentials for Account 1.
- **Upload CloudFormation template to S3:** Uploads the CloudFormation templates to an S3 bucket.
- **Deploy CloudFormation stack for Account 1:** Deploys the CloudFormation stack using the provided parameters.

\`\`\`yaml
  deploy-account-1:
    runs-on: ubuntu-latest
    needs: determine-changes
    if: needs.determine-changes.outputs.account_1_changed == 'true'
    steps:
      - name: Checkout code
        uses: actions/checkout@v2
        with:
          fetch-depth: 0

      - name: Set up AWS CLI for Account 1
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: \${{ secrets.AWS_ACCESS_KEY_ID_ACCOUNT_1 }}
          aws-secret-access-key: \${{ secrets.AWS_SECRET_ACCESS_KEY_ACCOUNT_1 }}
          aws-region: us-east-1

      - name: Upload CloudFormation template to S3
        run: |
          aws s3 cp ./modules/s3/s3-template.yaml s3://akin-ses-files/s3-template.yaml
          aws s3 cp ./modules/vpc/vpc-template.yaml s3://akin-ses-files/vpc-template.yaml

      - name: Deploy CloudFormation stack for Account 1
        run: |
          PARAMS=$(jq -r '.Parameters | to_entries | map("\(.key)=\(.value|tostring)") | join(" ")' account-1/parameters.json)
          echo "Deploying stack for Account 1 with parameters: ${PARAMS}"
          aws cloudformation deploy             --template-file account-1/main-template.yaml             --stack-name my-stack-account-1             --parameter-overrides ${PARAMS}
        shell: bash
\`\`\`

#### 3. Deploy to Account 2

This job runs only if changes are detected in the `account-2/` directory.

- **Checkout code:** Retrieves the latest code from the repository.
- **Set up AWS CLI for Account 2:** Configures AWS credentials for Account 2.
- **Upload CloudFormation template to S3:** Uploads the CloudFormation templates to an S3 bucket.
- **Deploy CloudFormation stack for Account 2:** Deploys the CloudFormation stack using the provided parameters.

\`\`\`yaml
  deploy-account-2:
    runs-on: ubuntu-latest
    needs: determine-changes
    if: needs.determine-changes.outputs.account_2_changed == 'true'
    steps:
      - name: Checkout code
        uses: actions/checkout@v2
        with:
          fetch-depth: 0

      - name: Set up AWS CLI for Account 2
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: \${{ secrets.AWS_ACCESS_KEY_ID_ACCOUNT_2 }}
          aws-secret-access-key: \${{ secrets.AWS_SECRET_ACCESS_KEY_ACCOUNT_2 }}
          aws-region: us-east-2
 
      - name: Upload CloudFormation template to S3
        run: |
          aws s3 cp ./modules/s3/s3-template.yaml s3://akin-ses-files-us-east-2/s3-template.yaml
          aws s3 cp ./modules/vpc/vpc-template.yaml s3://akin-ses-files-us-east-2/vpc-template.yaml

      - name: Deploy CloudFormation stack for Account 2
        run: |
          if [ -f account-2/parameters.json ]; then
            PARAMS=$(jq -r '.Parameters | to_entries | map("\(.key)=\(.value|tostring)") | join(" ")' account-2/parameters.json)
          else
            PARAMS=""
          fi
          echo "Deploying stack for Account 2 with parameters: ${PARAMS}"
          aws cloudformation deploy             --template-file account-2/main-template.yaml             --stack-name my-stack-account-2             --parameter-overrides ${PARAMS}
        shell: bash
\`\`\`

## Setup

1. **AWS Credentials:** Ensure that the secrets `AWS_ACCESS_KEY_ID_ACCOUNT_1`, `AWS_SECRET_ACCESS_KEY_ACCOUNT_1`, `AWS_ACCESS_KEY_ID_ACCOUNT_2`, and `AWS_SECRET_ACCESS_KEY_ACCOUNT_2` are set in your GitHub repository's secrets.

2. **S3 Buckets:** Ensure that the specified S3 buckets (`akin-ses-files` and `akin-ses-files-us-east-2`) exist and have the appropriate permissions.

3. **CloudFormation Templates:** Ensure that the CloudFormation templates and parameter files (`main-template.yaml`, `parameters.json`, etc.) are correctly set up in the respective directories (`account-1/` and `account-2/`).

## How to Use

- **Push Changes:** Push changes to the `main` branch that affect either the `account-1/` or `account-2/` directories.
- **Automatic Deployment:** The workflow will automatically determine if changes have been made to these directories and deploy the corresponding CloudFormation stacks to the respective AWS accounts.

This setup helps in maintaining and deploying infrastructure changes efficiently across multiple AWS accounts using GitHub Actions and AWS CloudFormation.
