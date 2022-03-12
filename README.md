# github-action-terraform

Reusable terraform github workflows that deploy to AWS and provides cost estimates on PRs.

<!-- BEGIN mktoc -->
- [Usage](#usage)
  - [Prerequisites](#prerequisites)
  - [Fmt](#fmt)
  - [Plan](#plan)
  - [Apply](#apply)
<!-- END mktoc -->

## Usage

### Prerequisites

These workflows assume you are deploying to AWS and you are using terraform workspaces. It supports multiple AWS accounts, each supporting multiple workspaces. They also assume you are using [Infracost](https://www.infracost.io/) and have a valid API key.

A certain directory structure is assumed. Each workspace should correspond with a tfvar files that follows this pattern `./tfvars/{{profile}}/{{workspace}}.tfvars`.

The backend file should also conform to `backends/{{profile}}.tfvars`.

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
    uses: rewindio/github-action-terraform/.github/workflows/fmt.yml@v1
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
    uses: rewindio/github-action-terraform/.github/workflows/plan.yml@v1
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
    uses: rewindio/github-action-terraform/.github/workflows/plan.yml@v1
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
    uses: rewindio/github-action-terraform/.github/workflows/apply.yml@v1
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
    uses: rewindio/github-action-terraform/.github/workflows/apply.yml@v1
    with:
      workspaces: ${{ needs.terraform-read-workspaces.outputs.production_workspaces }}
      profile: production
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.MY_AWS_ACCESS_KEY_ID_STAGING }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.MY_AWS_SECRET_ACCESS_KEY_STAGING }}
      GITHUB_PAT: ${{ secrets.MY_GITHUB_PAT }}

```
