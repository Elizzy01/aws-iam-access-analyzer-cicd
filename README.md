# Integrating AWS IAM Access Analyzer in a CI/CD Pipeline

## Overview

This project demonstrates how AWS IAM Access Analyzer can be integrated
into a CI/CD pipeline to automatically validate IAM policies before
infrastructure is deployed.

The pipeline uses GitHub as the source repository, AWS CodePipeline for
continuous delivery, AWS CodeBuild for validation and analysis, and AWS
CloudFormation for infrastructure deployment.

## Architecture

The CI/CD pipeline consists of three major stages:

1. **Source** — GitHub stores the CloudFormation template and supporting configuration.
2. **Validation & Analysis** — AWS CodeBuild runs CloudFormation linting and IAM policy validation using AWS IAM Access Analyzer.
3. **Deployment** — AWS CloudFormation deploys the template after the validation checks pass.

![AWS IAM Access Analyzer CI/CD Architecture](docs/screenshots/01-workshop-architecture.png)

## AWS Services

- AWS IAM Access Analyzer
- AWS CodePipeline
- AWS CodeBuild
- AWS CloudFormation
- GitHub

A hands-on AWS security and DevOps project that integrates **IAM Access Analyzer into a CI/CD pipeline** to automatically detect IAM policy issues in CloudFormation templates before deployment.

The project uses **AWS CodePipeline, AWS CodeBuild, IAM Access Analyzer, CloudFormation, Amazon S3, AWS IAM, AWS CodeConnections, and GitHub**.

---

## Project Overview

Infrastructure-as-Code enables rapid deployment of cloud resources, but IAM policy errors can introduce security vulnerabilities.

This project demonstrates a security-focused CI/CD workflow where CloudFormation templates are automatically:

1. Retrieved from GitHub.
2. Linted with `cfn-lint`.
3. Validated using `cfn-policy-validator`.
4. Analyzed using **IAM Access Analyzer**.
5. Blocked when critical IAM policy findings are detected.

### Pipeline Flow

```text
                         GitHub
                           │
                           ▼
                  AWS CodeConnections
                           │
                           ▼
                    AWS CodePipeline
                           │
              ┌────────────┴────────────┐
              │                         │
              ▼                         ▼
          Source Stage              Lint Stage
              │                         │
              │                    AWS CodeBuild
              │                      cfn-lint
              │                         │
              │                         ▼
              │                    Pass / Fail
              │
              ▼
       PolicyValidation Stage
              │
              ▼
         AWS CodeBuild
      cfn-policy-validator
              │
              ▼
      IAM Access Analyzer
              │
              ▼
       Security Findings
              │
              ▼
          Pass / Fail
```

---

## Architecture

![AWS IAM Access Analyzer CI/CD Architecture](docs/screenshots/01-workshop-architecture.png)

The pipeline uses an S3 artifact bucket to pass source artifacts between CodePipeline stages.

### AWS Services Used

| AWS Service                | Purpose                                           |
| -------------------------- | ------------------------------------------------- |
| **GitHub**                 | Source-code repository                            |
| **AWS CodeConnections**    | Secure connection between GitHub and CodePipeline |
| **AWS CodePipeline**       | CI/CD orchestration                               |
| **AWS CodeBuild**          | Executes linting and policy validation            |
| **IAM Access Analyzer**    | Analyzes IAM policies                             |
| **AWS CloudFormation**     | Infrastructure as Code                            |
| **Amazon S3**              | CodePipeline artifact storage                     |
| **AWS IAM**                | Service roles and permissions                     |
| **Amazon CloudWatch Logs** | CodeBuild execution logs                          |

---

## Repository Structure

```text
aws-iam-access-analyzer-cicd/
│
├── README.md
├── template.json
├── template-configuration.json
│
├── cloudformation/
│   └── template.yaml
│
├── docs/
│   └── screenshots/
│       └── 01-workshop-architecture.png
│
├── architecture/
│   └── .gitkeep
│
├── buildspec/
│   └── .gitkeep
│
└── iam/
    └── .gitkeep
```

---

## CloudFormation Template

The main validation target is:

```text
template.json
```

The template defines:

* An Amazon SQS queue
* An SQS queue policy
* An IAM role
* An IAM policy attached to the role

The template was intentionally modified during the workshop to demonstrate how IAM Access Analyzer identifies security and policy issues.

---

## Stage 1 — CloudFormation Linting

The first CodeBuild project is:

```text
cfn-lint
```

Its purpose is to detect CloudFormation template syntax and structural issues.

### Buildspec

```yaml
version: 0.2

phases:
  install:
    commands:
      - pip3 install cfn-lint

  build:
    commands:
      - cfn-lint -iW template.json
```

### CodeBuild Configuration

```text
Project: cfn-lint
Image: aws/codebuild/amazonlinux-x86_64-standard:5.0
Compute: BUILD_GENERAL1_SMALL
Source: NO_SOURCE
```

The actual source artifact is supplied by CodePipeline.

---

## Stage 2 — IAM Policy Validation

The second CodeBuild project is:

```text
cfn-policy-validator
```

This project runs AWS's CloudFormation policy validator.

### Buildspec

```yaml
version: 0.2

phases:
  install:
    commands:
      - pip3 install cfn-policy-validator

  build:
    commands:
      - cfn-policy-validator validate \
          --template-path template.json \
          --region us-east-1 \
          --template-configuration-file template-configuration.json
```

The validator uses **IAM Access Analyzer** to inspect IAM policies contained in the CloudFormation template.

---

# Deliberate Failure

A deliberate failure was part of the flow.

The original CloudFormation template contained an invalid IAM action:

```text
sqs:ThisActionDoesNotExist
```

IAM Access Analyzer detected the issue.

The resulting finding was:

```text
findingType: ERROR
code: INVALID_ACTION
message: The action sqs:ThisActionDoesNotExist does not exist.
```

This caused the PolicyValidation stage to fail.

This was important because it demonstrated that the pipeline was not simply checking whether the CloudFormation file was syntactically valid.

It was actually performing **IAM policy security analysis**.

---

## ⚠️ Additional Security Findings

IAM Access Analyzer also identified an external principal in the SQS queue policy.

The finding was:

```text
findingType: SECURITY_WARNING
code: EXTERNAL_PRINCIPAL
message: Resource policy allows access from external principals.
```

The external principal was:

```text
111122223333
```

The validator also identified a non-blocking warning because the queue policy did not initially specify a policy version:

```text
findingType: WARNING
code: MISSING_VERSION
```

The recommendation was to specify:

```json
"Version": "2012-10-17"
```

---

## Troubleshooting the Pipeline

The pipeline did not work perfectly on the first execution.

This was intentional and became an important part of the debugging process.

The initial pipeline state was:

```text
Source           Succeeded
Lint             Failed
PolicyValidation None
```

The next step was to inspect the CodeBuild execution.

---

## Investigating CodeBuild

The build was inspected using:

```bash
aws codebuild list-builds-for-project \
  --project-name cfn-lint \
  --region us-east-1 \
  --query 'ids[0:3]' \
  --output table
```

Then:

```bash
aws codebuild batch-get-builds \
  --ids BUILD_ID \
  --region us-east-1 \
  --query 'builds[0].{Status:buildStatus,Phases:phases[*].{Phase:phaseType,Status:phaseStatus,Context:phaseContext}}' \
  --output json
```

The build showed:

```text
SUBMITTED    SUCCEEDED
QUEUED       CLIENT_ERROR
COMPLETED    -
```

---

## Fixing CodeBuild IAM Permissions

The CodeBuild service role was inspected:

```text
codebuild-cfn-lint-service-role
```

Its trust relationship correctly allowed:

```text
codebuild.amazonaws.com
```

However, the role initially had no attached or inline permissions.

An inline policy was therefore added:

```text
cfn-lint-build-permissions
```

After the permission fix, the Lint stage succeeded.

---

## Policy Validator IAM Role

The policy validation CodeBuild project uses:

```text
codebuild-cfn-policy-validator-service-role
```

Its inline policy is:

```text
cfn-policy-validator-policy
```

The role was configured with permissions required by the validator, including:

```text
iam:GetPolicy
iam:GetPolicyVersion

access-analyzer:ListAnalyzers
access-analyzer:ValidatePolicy
access-analyzer:CreateAccessPreview
access-analyzer:GetAccessPreview
access-analyzer:ListAccessPreviewFindings
access-analyzer:CreateAnalyzer

s3:ListAllMyBuckets
cloudformation:ListExports
ssm:GetParameter
```

It also allows the required IAM service-linked role creation:

```text
iam:CreateServiceLinkedRole
```

with the condition:

```text
iam:AWSServiceName = access-analyzer.amazonaws.com
```

---

## Investigating the Policy Validation Failure

After fixing the CodeBuild permissions, the pipeline reached:

```text
Source           Succeeded
Lint             Succeeded
PolicyValidation Failed
```

The CodeBuild execution was then inspected.

The build successfully passed:

```text
QUEUED
PROVISIONING
DOWNLOAD_SOURCE
INSTALL
PRE_BUILD
```

but failed during:

```text
BUILD
```

This showed that CodeBuild itself was functioning correctly.

The failure was coming from the policy validation command.

---

## CloudWatch Logs

CloudWatch Logs were used to inspect the actual validator output.

Command:

```bash
aws logs get-log-events \
  --log-group-name /aws/codebuild/cfn-policy-validator \
  --log-stream-name BUILD_ID \
  --region us-east-1 \
  --query 'events[*].message' \
  --output text
```

The logs confirmed:

```text
Command did not exit successfully
cfn-policy-validator validate ...
exit status 2
```

The underlying reason was the IAM Access Analyzer finding:

```text
INVALID_ACTION
```

for:

```text
sqs:ThisActionDoesNotExist
```

---

## Fixing the CloudFormation Template

The invalid action was removed/replaced with a valid SQS action.

The queue policy was also corrected to include:

```json
"Version": "2012-10-17"
```

The original broken template was preserved locally as:

```text
template-broken.json
```

This provided a reference for the deliberate failure and debugging process.

---

## Git Commit

After correcting the template:

```bash
git add template.json
```

The changes were committed:

```bash
git commit -m "Fix IAM policy validation findings"
```

Commit:

```text
acf2870 Fix IAM policy validation findings
```

The changes were then pushed to GitHub:

```bash
git push
```

---

## Final Pipeline Execution

The GitHub push triggered a new CodePipeline execution.

Pipeline status was checked with:

```bash
aws codepipeline get-pipeline-state \
  --name iam-access-analyzer-cicd \
  --region us-east-1 \
  --query 'stageStates[*].{Stage:stageName,Status:latestExecution.status}' \
  --output table
```

The final result was:

```text
-----------------------------------
|        GetPipelineState         |
+-------------------+-------------+
|       Stage       |   Status    |
+-------------------+-------------+
|  Source           |  Succeeded  |
|  Lint             |  Succeeded  |
|  PolicyValidation |  Succeeded  |
+-------------------+-------------+
```

**All three pipeline stages succeeded.**

---

# 🔧 CodePipeline Configuration

Pipeline name:

```text
iam-access-analyzer-cicd
```

Region:

```text
us-east-1
```

Pipeline role:

```text
arn:aws:iam::970547354258:role/AWSCodePipelineServiceRole
```

Artifact bucket:

```text
iam-access-analyzer-cicd-a2cicdartifactbucket-0yrjpwrhpdyv
```

Pipeline stages:

```text
Source
Lint
PolicyValidation
```

---

## Source Stage

Provider:

```text
CodeStarSourceConnection
```

Repository:

```text
Elizzy01/aws-iam-access-analyzer-cicd
```

Branch:

```text
main
```

The GitHub repository is connected through AWS CodeConnections.

---

## Lint Stage

Provider:

```text
AWS CodeBuild
```

Project:

```text
cfn-lint
```

Purpose:

```text
CloudFormation template linting
```

---

## PolicyValidation Stage

Provider:

```text
AWS CodeBuild
```

Project:

```text
cfn-policy-validator
```

Purpose:

```text
IAM policy validation using IAM Access Analyzer
```

---

### Useful AWS CLI Commands

### Check Pipeline Status

```bash
aws codepipeline get-pipeline-state \
  --name iam-access-analyzer-cicd \
  --region us-east-1 \
  --query 'stageStates[*].{Stage:stageName,Status:latestExecution.status}' \
  --output table
```

### List CodeBuild Builds

```bash
aws codebuild list-builds-for-project \
  --project-name cfn-policy-validator \
  --region us-east-1 \
  --query 'ids[0:3]' \
  --output table
```

### Inspect a Build

```bash
aws codebuild batch-get-builds \
  --ids BUILD_ID \
  --region us-east-1 \
  --query 'builds[0].{Status:buildStatus,Phases:phases[*].{Phase:phaseType,Status:phaseStatus,Context:phaseContext}}' \
  --output json
```

### Inspect CloudWatch Logs

```bash
aws logs get-log-events \
  --log-group-name /aws/codebuild/cfn-policy-validator \
  --log-stream-name BUILD_ID \
  --region us-east-1 \
  --query 'events[*].message' \
  --output text
```

### Check IAM Role Policies

```bash
aws iam list-attached-role-policies \
  --role-name ROLE_NAME \
  --output table
```

For inline policies:

```bash
aws iam list-role-policies \
  --role-name ROLE_NAME \
  --output table
```

---

# 📚 Key Lessons Learned

## CI/CD

* How AWS CodePipeline
