# GitHub Integration - Direct IacService Mode

A CI/CD integration solution based on GitHub Actions and Alibaba Cloud IacService, which directly triggers Terraform stack automated deployment through IacService API.

> **🌐 Language**: [中文文档](README-CN.md) | [English Docs](README.md)

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
  - [Prepare Code Repository](#prepare-code-repository)
  - [Initialize Cloud Infrastructure](#initialize-cloud-infrastructure)
  - [GitHub Configuration](#github-configuration)
- [GitHub Actions Workflow Configuration](#github-actions-workflow-configuration)
  - [Workflow Files](#workflow-files)
  - [Workflow Dependencies](#workflow-dependencies)
- [Daily Usage](#daily-usage)
  - [Create IacService Stack](#create-iacservice-stack)
  - [Configure Secret Parameters](#configure-secret-parameters)
  - [Change Process](#change-process)
  - [Running Parameters](#running-parameters)
- [Operations Reference](#operations-reference)
  - [Troubleshooting](#troubleshooting)
  - [CI Dependencies](#ci-dependencies)
  - [Multi-Environment Configuration](#multi-environment-configuration)
- [Appendix](#appendix)
  - [Workflow Design Details](#workflow-design-details)

---

# Overview

This document details the complete operation process for configuring and managing stacks when GitHub serves as the Version Control System (VCS).

> **Prerequisite**: This document assumes you have completed preliminary preparation and IacService configuration. For basic configuration, please refer to the VCS-driven Stack CI/CD Operation Guide.

The core advantages of GitHub integration are:
- Leverage GitHub Actions as the CI/CD engine
- Define automation processes through Workflow files
- Deep integration with GitHub's native Pull Request, Code Review, Branch Protection, and other collaboration features
- Achieve a complete closed loop of "automatic deployment triggered upon code review approval"

```
Developer → PR Comment → GitHub Actions → IacService API(Upload Code + Trigger Execution) → Deploy → IacService API(Query Result) → GitHub Actions → PR Comment
```

The above diagram shows the complete operation link from GitHub to manage IacService stacks. The interaction between GitHub Actions and IacService is accomplished through workflows for code file upload, task triggering, and execution log retrieval.

Workflows are triggered by PR comments and support the following commands:

```
iac terraform plan [-profile=<profile>] [-stack=<stack>]
iac terraform apply [-profile=<profile>] [-stack=<stack>]
```

> **Auto-detection**: When a PR contains only changes to the `deployments/` directory, the `-profile` and `-stack` parameters can be omitted, and the workflow will automatically infer them from the changed files. If the PR contains changes to other directories, these parameters must be manually specified.

Helper scripts (`scripts/`):

| Script | Language | Dependencies | Purpose |
|--------|----------|--------------|---------|
| `upload_iac_module.py` | Python 3.12 | `alibabacloud_iacservice20210806` | Packages and uploads code to IacService module. Reads credentials and configuration from environment variables `IAC_ACCESS_KEY_ID`/`IAC_ACCESS_KEY_SECRET`/`IAC_REGION`/`CODE_MODULE_ID` |
| `trigger_stack.py` | Python 3.12 | `alibabacloud_iacservice20210806` | Triggers Plan or Apply operations on a Stack, returns Trigger ID for subsequent queries |
| `get_trigger_result.py` | Python 3.12 | `alibabacloud_iacservice20210806`, `PyYAML` | Polls IacService API for execution results with formatted output. Supports multi-profile parallel polling (multi-threaded), default timeout 600 seconds |

---

# Prerequisites

## Prepare Code Repository

Create a new repository in GitHub, copy all files from the `ci-templates/direct-iacservice/github/` directory of the IacService Stack multi-account management scaffold project to the root directory of your repository, then commit and push:

```bash
git add .
git commit -m "init"
git push --set-upstream origin main
```

---

## Initialize Cloud Infrastructure

### 1. Create RAM User 1: Management User

Used for one-time initialization configuration of IacService resources. Subsequent operations can be performed by operations personnel using this user account:

1. Log in to the [RAM Console](https://ram.console.aliyun.com/), create a user and enable [OpenAPI Access]
2. Attach the following policies to the user:
   - `AliyunRAMFullAccess`
   - Custom policy `IaCServiceStackFullAccess` (policy content see [docs/iam-policies.md](../../../docs/iam-policies.md))

### 2. Create RAM User 2: Pipeline Trigger User

Used for GitHub Actions to call IacService API. Configure the AccessKey to GitHub Secrets:

1. Create a user and enable [OpenAPI Access]
2. Attach custom policy `IaCServiceStackTriggerAccess` (policy content see [docs/iam-policies.md](../../../docs/iam-policies.md))
3. Create an AccessKey, record the AccessKey ID and Secret for later GitHub Secrets configuration (this document uses development environment as an example, assuming the key pair is named `DEV_ACCESS_KEY_ID` / `DEV_ACCESS_KEY_SECRET`)

### 3. Create RAM Role: IacService Execution Role

Used to authorize IacService to assume this role to execute Terraform templates:

1. Create a new RAM role in the RAM console
2. Configure the trust policy to allow IacService to assume this role (trust policy content see [docs/iam-policies.md](../../../docs/iam-policies.md))
3. Attach the permission policies required to execute Terraform templates (e.g., if the template contains ECS instances, attach ECS-related permissions)

> **Multi-Environment Note**: If you have multiple environments (dev, staging, prod, etc.), it is recommended to create different RAM Roles to achieve complete isolation between environments.

### 4. Create IacService Template

This step creates the IacService template. A template corresponds to a piece of code. Each code modification needs to be republished as a new version. Different changes to the same code correspond to different template versions.

> **Important**: Please record the template ID (ModuleId) after creation, as it will be needed in subsequent steps.

**Step 1: Create New Template**

Log in to [Alibaba Cloud IacService](https://iac.console.aliyun.com/), click [Template Management], and select [Create Template].

**Step 2: Select Template Source**

Select [Blank Template] method.

**Step 3: Fill in Template Parameters**

Fill in the following parameters and click [Submit]:

| Parameter | Required | Description |
|-----------|----------|-------------|
| Template Name | Yes | The name of the template, must be unique |
| Template Description | No | Description of the template |
| Add to Project Group | No | Optional project group for unified template management |
| Tags | No | Tags for the template for easier classification |

Fill the created `moduleId` into the `code_module_id` value in `deployments/[env]/profile.yaml`:

```yaml
code_module_id: "mod-xxx"
```

> **Multi-Environment Note**: If you have multiple environments (dev, staging, prod, etc.), you need to perform independent configuration for each environment using different IacService templates to achieve complete isolation between environments.

---

## GitHub Configuration

### 1. Create RAM User and Configure IacService Permissions

> **Note**: If you have already created a "Pipeline Trigger User" in the "Initialize Cloud Infrastructure" step, you can directly use that user's AccessKey and skip this step.

Create a RAM User that only allows API access, create an AccessKey, and grant the following minimum permissions:

```json
{
  "Version": "1",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "iacservice:UploadModule",
      "Resource": "acs:iacservice:*:*:module/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "iacservice:GetStackExecutionResult",
        "iacservice:TriggerStackExecution"
      ],
      "Resource": "acs:iacservice:*:*:stack/*"
    }
  ]
}
```

### 2. Configure GitHub Secrets

Add the AccessKey to the GitHub repository Secrets, storing each Key and Value separately. The Key name should correspond to the key pair name configured in `profile.yaml`:

| Secrets Key | Secrets Value |
|-------------|---------------|
| `DEV_ACCESS_KEY_ID` | `"xxx"` |
| `DEV_ACCESS_KEY_SECRET` | `"xxx"` |

> If you have multiple environments, configure them by environment prefix, e.g., `STAGING_ACCESS_KEY_ID`, `STAGING_ACCESS_KEY_SECRET`, `PROD_ACCESS_KEY_ID`, `PROD_ACCESS_KEY_SECRET`

### 3. Declare Workflow Environment Variables

Declare Secrets references in the `env` block of `.github/workflows/pull-request-comment.yml` and `scheduled-check.yml`:

```yaml
env:
  DEV_ACCESS_KEY_ID: ${{ secrets.DEV_ACCESS_KEY_ID }}
  DEV_ACCESS_KEY_SECRET: ${{ secrets.DEV_ACCESS_KEY_SECRET }}
  # Other environments...
```

> **Note**: All workflow files (including shared workflows) that use hardcoded environment prefixes (e.g., `DEV_`, `PROD_`) must also add corresponding Secrets references.

### 4. Initialize IacService Template

Submit the current branch code and manually trigger the **Scheduled Check** workflow in GitHub Actions to initialize the IacService template (select the current branch before running, no need to check "Whether to run detect check").

> If an error occurs, it may be because multiple environments exist under `deployments/` but not all are configured. Delete extra environment directories or complete the configuration.

---

# GitHub Actions Workflow Configuration

The repository contains 5 workflow files, divided into two categories:

## Workflow Files

**Main Workflows** (directly triggered by GitHub events):

| Workflow | Trigger | Function |
|----------|---------|----------|
| `pull-request-comment.yml` | PR comment created/edited | Parses `iac terraform plan/apply` commands, packages and uploads code, triggers IacService execution, writes results back to PR |
| `scheduled-check.yml` | Manual trigger / Scheduled task | Uploads source packages for all environments to IacService template and publishes as versions, optionally triggers drift detection |

**Shared Workflows** (called by main workflows, can be independently extracted and reused):

| Workflow | Caller | Function |
|----------|--------|----------|
| `shared-ci-get-pull-request-info.yml` | `pull-request-comment` | Validates PR mergeability, parses `-profile`/`-stack` parameters from comments, auto-infers affected stacks from changed files |
| `shared-ci-upload-source-package.yml` | `scheduled-check` | Iterates through all profiles, executes `make build-package` to build source packages and upload to IacService template versions; single profile failure doesn't affect others |
| `shared-ci-upload-trigger-file.yml` | `pull-request-comment` | Builds source packages for each profile, calls IacService API with execution commands and changed stacks to trigger execution; fails fast on any profile failure |

## Workflow Dependencies

```
pull-request-comment.yml
├── shared-ci-get-pull-request-info.yml    # Step 1: Parse commands, get PR info
├── shared-ci-upload-trigger-file.yml      # Step 2: Upload code as template version + Trigger execution
└── get_exec_result (inline job)            # Step 3: Poll results → Write back to PR Comment

scheduled-check.yml
├── shared-ci-upload-source-package.yml    # Step 1: Upload all profile source packages
└── get_exec_result (inline job)            # Step 2: Poll detection results (only when run_detect=true)
```

---

# Daily Usage

## Create IacService Stack

Before executing code, you need to create the corresponding stack on Alibaba Cloud IacService to host the code execution. After the repository code initialization is complete, first use the Scheduled Check workflow to upload the code as an IacService template version, then create the corresponding stack on IacService.

Go to Alibaba Cloud IacService (https://iac.console.aliyun.com/stack) to create a Stack:

**Step 1: Create Stack**

Click [Stack], select [Create Stack].

**Step 2: Fill in Stack Information**

| Parameter | Required | Description |
|-----------|----------|-------------|
| Stack Name | Yes | The name of the stack, must be unique |
| Description | No | Description of the stack |
| Stack Code Source | Yes | **Module**: Select template from IacService created in Prerequisites<br>⚠️ **Note**: Direct mode only supports Module source, OSS source is not available |
| Template ID/Version | Yes | Select the template created in Prerequisites |
| Working Directory | Yes | The path where the stack configuration file is stored in the code, e.g., `stacks/demo`                                                                                    |
| RAM Role | Yes | Select the role created in Prerequisites for running Terraform templates |

**Step 3: Associate Parameter Set**

Select or create a parameter set, click [Next].

> **Secret Parameters**: Parameter sets support defining secret parameters (e.g., passwords, API Tokens, access keys). For details, see the [Secret Parameters Guide](../../../docs/secret-parameters.md).

**Step 4: Confirm Creation**

After checking the configuration information is correct, click [Create].

> **Batch Creation**: It is recommended to create all Stacks defined in the `stacks/` directory at once. Each Stack corresponds to an independent stack. After creation, the first `plan` will be automatically executed to verify the correctness of the configuration.

---

## Configure Secret Parameters

Parameter sets support defining secret parameters for securely storing passwords, API Tokens, access keys, and other confidential information. Secret values are automatically encrypted via Alibaba Cloud KMS and masked across the console UI, execution logs, API responses, and PR comment results.

Basic workflow for using secret parameters:

1. **Set up encryption key**: Navigate to **System Settings > Encryption Settings**, select a Service Key (auto-created) or User-Managed Key (created in KMS).
2. **Create secret parameters**: When adding a parameter in a parameter set, check the **Secret** option and save.
3. **Declare component variables**: In the Stack Component, variables receiving secret values must declare `sensitive: true`.
4. **Reference the parameter set**: In Deployment configuration, reference the parameter set via the `store` block using the format `store.varset.<STORE_NAME>.<VARIABLE_NAME>`.

For detailed instructions, see the [Secret Parameters Guide](../../../docs/secret-parameters.md).

---

## Change Process

1. Create a development branch, commit code changes, push to GitHub

```bash
git checkout -b dev
# Modify configurations in stacks/ or deployments/[env]/...
git add .
git commit -m "update stack config"
git push --set-upstream origin dev
```

2. Create a PR to the main branch (e.g., `main`), enter commands in PR comments to trigger deployment:

```bash
iac terraform plan -profile=dev -stack=demo
iac terraform apply -profile=dev -stack=demo
```

3. Execution results are automatically written back to PR comments — confirm apply results meet expectations

4. After review approval, merge changes to the main branch

---

## Running Parameters

Runtime parameters are passed via PR comments in the following format:

```
iac terraform plan/apply [-profile=<profileName>] [-stack=<StackName>]
```

**Parameter Reference**:

| Parameter | Required | Description |
|-----------|----------|-------------|
| `-profile` | Yes* | Execution environment, corresponds to environment name under `deployments/` directory |
| `-stack` | Yes* | Target stack path, supports directory nesting, e.g., `demo/subDir` |

> *When a PR only contains changes in the `deployments/` directory, both parameters can be omitted — the workflow will auto-infer from changed files.

**Notes**:

- A PR can be reused until merged, but creating independent PRs for each task is recommended for traceability
- `plan` must be executed before each `apply`; `plan` can be executed multiple times consecutively
- Different environment `-profile` values correspond to different Alibaba Cloud accounts and resources, with no mutual impact

---

# Operations Reference

## Troubleshooting

| Issue | Resolution |
|-------|-----------|
| No deployment response | Check if GitHub Secrets are configured correctly, if IacService is operational, if RAM permissions are sufficient |
| Command not recognized | Confirm PR Comment format is `iac terraform plan/apply`, note the prefix must match exactly |
| `make build-package` fails | Confirm the repository root contains a Makefile with the `build-package` target |
| Profile upload fails | Check if `access_key_id`/`access_key_secret` in `deployments/[env]/profile.yaml` match the Key names in GitHub Secrets |
| Code upload fails | Check if `code_module_id` is correct, if AK has IacService permissions |
| Trigger execution fails | Check if `code_module_version` is correct, if the Stack has been created |
| Result retrieval timeout | Default polling timeout is 600 seconds, check IacService console to confirm if the Stack is executing |
| Partial multi-environment failure | Check GitHub Actions logs, `shared-ci-upload-source-package` will list the failed profile names |
| View detailed errors | Check GitHub Actions run logs, or check execution records in IacService console |
| Secret parameter decryption failure | Verify encryption settings are initialized (System Settings > Encryption Settings) and the KMS key is enabled and not deleted or disabled |

---

## CI Dependencies

GitHub Actions runtime requires the following dependencies:

| Dependency | Purpose | Provisioning |
|------------|---------|-------------|
| **Python 3.12** | Running `upload_iac_module.py`, `trigger_stack.py`, and `get_trigger_result.py` scripts | Auto-installed via `actions/setup-python@v5` |
| **alibabacloud_iacservice20210806** | Alibaba Cloud IacService SDK for uploading code packages, triggering execution, and retrieving execution results | `pip install alibabacloud_iacservice20210806` |
| **PyYAML** | Parsing `profile.yaml` and credential files | `pip install PyYAML` |
| **make** | Running `make build-package` to build source packages | Pre-installed in `ubuntu-latest` image |
| **curl / jq** | Fetching PR info, parsing JSON responses | Pre-installed in `ubuntu-latest` image |

> **Offline Environment Note**: If GitHub Actions runs on self-hosted Runners without public internet access, pre-install the above dependencies in the Runner image or cache dependency packages in a private repository/artifact store.

---

## Multi-Environment Configuration

If you need to add a new Alibaba Cloud account (e.g., add a `test` environment), complete the following configuration:

### 1. Create RAM Role and IacService Template

Refer to [Create RAM Role](../../../docs/iam-policies.md) and [Create IacService Template](#4-create-iacservice-template) chapters to create independent RAM Role and template for the new environment.

### 2. Create deployments Configuration

```bash
cp -r deployments/dev deployments/test
cd deployments/test
```

Modify `code_module_id` in `profile.yaml` to the `moduleId` from Step 1, and add the AK variable name in `profile.yaml` as a new GitHub Secrets Key.

### 3. Configure GitHub Secrets

Add the new account in GitHub repository Settings → Secrets and variables → Actions:

| Secrets Key | Secrets Value |
|-------------|---------------|
| `TEST_ACCESS_KEY_ID` | New account's AccessKey ID |
| `TEST_ACCESS_KEY_SECRET` | New account's AccessKey Secret |

### 4. Declare CI Environment Variables

Add the new account reference in the `env` block of `.github/workflows/pull-request-comment.yml` and `scheduled-check.yml`:

```yaml
env:
  # ... Other environments
  TEST_ACCESS_KEY_ID: ${{ secrets.TEST_ACCESS_KEY_ID }}
  TEST_ACCESS_KEY_SECRET: ${{ secrets.TEST_ACCESS_KEY_SECRET }}
```

> **Note**: All workflow files (including shared workflows) that use hardcoded environment prefixes (e.g., `DEV_`, `PROD_`) must also add corresponding `TEST_` references.

### 5. Create Stack

Go to Alibaba Cloud Iac console to create a Stack, selecting the `moduleId` and RAM Role created in Step 1. Refer to [Create IacService Stack](#create-iacservice-stack) chapter.

### 6. Verify Configuration

After submitting the configuration, manually trigger the **Scheduled Check** workflow to confirm that the new environment's source package upload is successful.

> **Account Isolation Recommendation**: Each Alibaba Cloud account corresponds to independent RAM Role and IacService template resources. Manage credentials through different GitHub Secrets to achieve complete isolation. It is recommended to divide different environments into different Alibaba Cloud master accounts and use Alibaba Cloud Resource Directory for unified governance.

---

# Appendix

## Workflow Design Details

### pull-request-comment.yml

**Trigger Condition**: When a PR comment is created or edited (`issue_comment: [created, edited]`)

**Workflow**:

1. **get_trigger_info** — Calls `shared-ci-get-pull-request-info.yml` to parse commands in comments, get PR head SHA and base ref, infer affected stacks
2. **process_trigger_file** — Calls `shared-ci-upload-trigger-file.yml` to package code and upload to IacService template, trigger execution
3. **get_exec_result** — Polls IacService API to get execution results, formats and writes back to PR comment

**Supported Command Format**:

```
iac terraform plan [-profile=<profile>] [-stack=<stack>]
iac terraform apply [-profile=<profile>] [-stack=<stack>]
```

> When a PR contains only changes to the `deployments/` directory, the workflow will automatically infer profiles and stacks from the changed files. At this time, `-profile` and `-stack` parameters are optional. If the PR contains changes to other directories, these parameters must be manually specified.

### scheduled-check.yml

**Trigger Condition**: Manually triggered (`workflow_dispatch`) or scheduled to run automatically

**Workflow**:

1. **process_code** — Calls `shared-ci-upload-source-package.yml` to process source package uploads for all profiles (`profiles: all`); if `run_detect` is enabled, additionally triggers detection
2. **get_exec_result** — Executes only when `run_detect=true` or scheduled trigger, gets and outputs execution results

**Main Purposes**:

- **Drift Detection**: Triggered by `run_detect=true`, automatically detects deviations between actual cloud resources and Terraform configurations
- **Source Package Initialization**: Uploads source packages to IacService template during initial configuration, preparing for PR deployment
- **Code Integrity Verification**: Periodically verifies that code for all environments can be built normally

> **Code Source**: By default, uploads code from the `main` branch. Drift detection compares cloud resource status with `main` branch configuration and generates a detection report if deviations are found.

### Shared Workflows

**shared-ci-get-pull-request-info.yml**

Gets detailed PR information and parses command parameters. Main functions: validate PR mergeability, extract head SHA and base ref, parse `-profile`/`-stack` parameters, auto-infer affected stacks from changed files.

Main outputs: `command`, `base_ref`, `head_sha`, `all_changed_stacks`

**shared-ci-upload-source-package.yml**

Builds source packages and uploads them to IacService template, supporting multi-profile batch processing. Executes `make build-package PROFILE=<profile>` for each profile to build code packages, uploads to IacService template via `upload_iac_module.py`. Single profile failure does not affect other profiles; failure information is summarized after all processing is complete.

**shared-ci-upload-trigger-file.yml**

Uploads source packages to IacService template and triggers execution API to start IacService execution. Parses `all_changed_stacks` parameters and performs the following operations for each profile:

1. Build source package
2. Call `upload_iac_module.py` to upload to IacService template, get `version_id`
3. Call `trigger_stack.py` to trigger Plan or Apply execution, get `trigger_id`

Uses fail-fast strategy; immediately terminates on any profile failure.

Trigger information example:

```
Profile: dev, Version ID: v123456
Profile: dev, Trigger ID: trg-abc123...
```
