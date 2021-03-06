# github-action-terraform-aws

Reusable terraform github workflows that deploy to AWS, provides cost estimates on PRs and static anaylsis using Checkov.

<!-- BEGIN mktoc -->
- [Usage](#usage)
  - [Prerequisites](#prerequisites)
  - [Fmt](#fmt)
  - [Checkov](#checkov)
  - [Plan](#plan)
  - [Apply](#apply)
  - [State Unlock](#state-unlock)
<!-- END mktoc -->

## Usage

### Prerequisites

These workflows assume you are deploying to AWS and you are using terraform workspaces. It supports multiple AWS accounts, each supporting multiple workspaces. They also assume you are using [Infracost](https://www.infracost.io/) and have a valid API key.

A certain directory structure is assumed. Each workspace should correspond with a tfvar files that follows this pattern `./tfvars/{{profile}}/{{workspace}}.tfvars`.

The backend file should also conform to `backend/{{profile}}.tfvars`.

A `.terraform-version` file is required in the root of repo. This is a convention used by both [tfenv](https://github.com/tfutils/tfenv) and [tfswitch](https://github.com/warrensbox/terraform-switcher).

### Fmt


```yaml
# .github/workflows/tf-fmt.yml
name: terraform-fmt
on: pull_request
concurrency: terraform-fmt

jobs:
  terraform-fmt:
    name: "Terraform"
    uses: rewindio/github-action-terraform-aws/.github/workflows/fmt.yml@v1
```

### Checkov

Static analysis using Checkov can be performed by including `checkov.yml` into the workflow

```yaml
# .github/workflows/checkov.yml

name: checkov
concurrency: checkov
on: pull_request

jobs:
  checkov-static-analysis:
    name: "checkov"
    uses: rewindio/github-action-terraform-aws/.github/workflows/checkov.yml@v1
    secrets:
      GITHUB_PAT: ${{ secrets.MY_GITHUB_PAT }}
```

### Plan

Plans are run when a pull request is open.

Within the `terraform-plan` workflow, Infracost is ran to provide a cost estimate in the pull request.

The optional parameter `infracost_usage_file` is a YAML file that contains usage estimates for usage-based resources.


```yaml
# .github/workflows/tf-plan.yml

name: terraform-plan
on: pull_request
concurrency: terraform

jobs:
  terraform-read-workspaces:
    name: "Read Workspaces"
    runs-on: ubuntu-latest
    outputs:
      staging_workspaces: ${{ steps.set-workspaces.outputs.staging-workspaces }}
      production_workspaces: ${{ steps.set-workspaces.outputs.production-workspaces }}
    steps:
    - name: checkout
      uses: actions/checkout@v2
      with:
        fetch-depth: 1
    - id: set-workspaces
      run: |
        echo "::set-output name=staging-workspaces::$(find tfvars/staging -name "*.tfvars" -not -type d -exec basename {} \; | cut -d '.' -f1  | jq -R . | jq -cs)"
        echo "::set-output name=production-workspaces::$(find tfvars/production -name "*.tfvars" -not -type d -exec basename {} \; | cut -d '.' -f1  | jq -R . | jq -cs)"

  terraform-plan-staging:
    name: "Terraform"
    needs: terraform-read-workspaces
    uses: rewindio/github-action-terraform-aws/.github/workflows/plan.yml@v1
    with:
      workspaces: ${{ needs.terraform-read-workspaces.outputs.staging_workspaces }}
      profile: staging
      infracost_usage_file: infracost.yml # OPTIONAL parameter
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.MY_AWS_ACCESS_KEY_ID_STAGING }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.MY_AWS_SECRET_ACCESS_KEY_STAGING }}
      GITHUB_PAT: ${{ secrets.MY_GITHUB_PAT }}
      INFRACOST_API_KEY: ${{ secrets.INFRACOST_API_KEY }}

  terraform-plan-production:
    name: "Terraform"
    needs: terraform-read-workspaces
    uses: rewindio/github-action-terraform-aws/.github/workflows/plan.yml@v1
    with:
      workspaces: ${{ needs.terraform-read-workspaces.outputs.production_workspaces }}
      profile: production
      infracost_usage_file: infracost.yml # OPTIONAL parameter
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.MY_AWS_ACCESS_KEY_ID_STAGING }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.MY_AWS_SECRET_ACCESS_KEY_STAGING }}
      GITHUB_PAT: ${{ secrets.MY_GITHUB_PAT }}
      INFRACOST_API_KEY: ${{ secrets.INFRACOST_API_KEY }}
```

### Apply

Applies are run when there is a `terraform apply` comment made on the pull request.

The pull request must be approved for an apply to begin.

```yaml
# .github/workflows/tf-apply.yml

name: terraform-apply
on: pull_request
concurrency: terraform

jobs:
  terraform-read-workspaces:
    name: "Read Workspaces"
    runs-on: ubuntu-latest
    outputs:
      staging_workspaces: ${{ steps.set-workspaces.outputs.staging-workspaces }}
      production_workspaces: ${{ steps.set-workspaces.outputs.production-workspaces }}
    steps:
    - name: checkout
      uses: actions/checkout@v2
      with:
        fetch-depth: 1
    - id: set-workspaces
      run: |
        echo "::set-output name=staging-workspaces::$(find tfvars/staging -name "*.tfvars" -not -type d -exec basename {} \; | cut -d '.' -f1  | jq -R . | jq -cs)"
        echo "::set-output name=production-workspaces::$(find tfvars/production -name "*.tfvars" -not -type d -exec basename {} \; | cut -d '.' -f1  | jq -R . | jq -cs)"

  terraform-apply-staging:
    name: "Terraform"
    needs: terraform-read-workspaces
    uses: rewindio/github-action-terraform-aws/.github/workflows/apply.yml@v1
    with:
      workspaces: ${{ needs.terraform-read-workspaces.outputs.staging_workspaces }}
      profile: staging
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.MY_AWS_ACCESS_KEY_ID_STAGING }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.MY_AWS_SECRET_ACCESS_KEY_STAGING }}
      GITHUB_PAT: ${{ secrets.MY_GITHUB_PAT }}

  terraform-apply-production:
    name: "Terraform"
    needs: terraform-read-workspaces
    uses: rewindio/github-action-terraform-aws/.github/workflows/apply.yml@v1
    with:
      workspaces: ${{ needs.terraform-read-workspaces.outputs.production_workspaces }}
      profile: production
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.MY_AWS_ACCESS_KEY_ID_STAGING }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.MY_AWS_SECRET_ACCESS_KEY_STAGING }}
      GITHUB_PAT: ${{ secrets.MY_GITHUB_PAT }}

```

### State Unlock

Sometimes state gets locked due to the terraform process ending abruptly (i.e user cancelation or a runner issue).

This workflow attempts to "brute-force" unlock the state for all workspaces in the profile with the provided lock_id.

```yaml
# .github/workflows/tf-state-ulnock.yml
name: terraform-state-unlock

concurrency: terraform

on: 
  workflow_dispatch:
    inputs:
      profile:
        description: 'The aws profile to use (staging/production)' 
        required: true
        default: 'staging'
      lock_id:
        description: 'The terraform lock id in the error' 
        required: true

jobs:

  terraform-read-workspaces:
    name: "Read Workspaces"
    runs-on: ubuntu-latest
    outputs:
      workspaces: ${{ steps.set-workspaces.outputs.workspaces }}
    steps:
    - name: checkout
      uses: actions/checkout@v2
      with:
        fetch-depth: 1
    - id: set-workspaces
      run: |
        echo "::set-output name=workspaces::$(find tfvars/${{ github.events.inputs.profile }} -name "*.tfvars" -not -type d -exec basename {} \; | cut -d '.' -f1  | jq -R . | jq -cs)"

  terraform-state-unlock-staging:
    name: "Unlock"
    uses: rewindio/github-action-terraform-aws/.github/workflows/state-unlock.yml@v1
    needs: terraform-read-workspaces
    if: github.event.inputs.profile == 'staging'
    with:
      profile: "${{ github.event.inputs.profile }}"
      lock_id: "${{ github.event.inputs.lock_id }}"
      workspaces: ${{ needs.terraform-read-workspaces.outputs.workspaces }}
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.MY_AWS_ACCESS_KEY_ID_STAGING }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.MY_AWS_SECRET_ACCESS_KEY_STAGING }}
      GITHUB_PAT: ${{ secrets.MY_GITHUB_PAT }}

  terraform-state-unlock-production:
    name: "Unlock"
    uses: rewindio/github-action-terraform-aws/.github/workflows/state-unlock.yml@v1
    needs: terraform-read-workspaces
    if: github.event.inputs.profile == 'production'
    with:
      profile: "${{ github.event.inputs.profile }}"
      lock_id: "${{ github.event.inputs.lock_id }}"
      workspaces: ${{ needs.terraform-read-workspaces.outputs.workspaces }}
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.MY_AWS_ACCESS_KEY_ID_PRODUCTION }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.MY_AWS_SECRET_ACCESS_KEY_PRODUCTION }}
      GITHUB_PAT: ${{ secrets.MY_GITHUB_PAT }}
```
