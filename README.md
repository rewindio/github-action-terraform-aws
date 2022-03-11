# github-action-terraform

Reusable terraform github workflows.

<!-- BEGIN mktoc -->
- [Usage](#usage)
  - [Config](#config)
  - [Fmt](#fmt)
  - [Plan](#plan)
  - [Apply](#apply)
<!-- END mktoc -->

## Usage

### Config

A `terraform-config.yml` is required so that workspaces can be shared between `terraform-plan` and `terraform-apply`.

```yaml
---
profiles:
  staging:
    workspaces:
      - foo1
      - foo2
  production:
    workspaces:
      - foo3
      - foo4
```

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
  read-terraform-config:
    runs-on: ubuntu-latest
    outputs:
      staging_config: ${{ steps.set-config.outputs.staging-config }}
      production_config: ${{ steps.set-config.outputs.production-config }}
    steps:
    - name: checkout
      uses: actions/checkout@v2
      with:
        fetch-depth: 1
    - name: yq - portable yaml processor
      uses: mikefarah/yq@v4.21.1
    - id: set-config
      run: |
        echo "::set-output name=staging-config::$(yq -o json -I 0 '.profiles.staging.workspaces' terraform-config.yml)"
        echo "::set-output name=production-config::$(yq -o json -I 0 '.profiles.production.workspaces' terraform-config.yml)"

  terraform-plan-staging:
    name: "Terraform"
    needs: read-terraform-config
    uses: rewindio/github-action-terraform/.github/workflows/plan.yml@v1
    with:
      config: ${{ needs.read-terraform-config.outputs.staging_config }}
      profile: staging
      infracost_usage_file: infracost.yml # OPTIONAL parameter
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.MY_AWS_ACCESS_KEY_ID_STAGING }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.MY_AWS_SECRET_ACCESS_KEY_STAGING }}
      GITHUB_PAT: ${{ secrets.MY_GITHUB_PAT }}
      INFRACOST_API_KEY: ${{ secrets.INFRACOST_API_KEY }}

  terraform-plan-production:
    name: "Terraform"
    needs: read-terraform-config
    uses: rewindio/github-action-terraform/.github/workflows/plan.yml@v1
    with:
      config: ${{ needs.read-terraform-config.outputs.production_config }}
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
  read-terraform-config:
    runs-on: ubuntu-latest
    outputs:
      staging_config: ${{ steps.set-config.outputs.staging-config }}
      production_config: ${{ steps.set-config.outputs.production-config }}
    steps:
    - name: checkout
      uses: actions/checkout@v2
      with:
        fetch-depth: 1
    - name: yq - portable yaml processor
      uses: mikefarah/yq@v4.21.1
    - id: set-config
      run: |
        echo "::set-output name=staging-config::$(yq -o json -I 0 '.profiles.staging.workspaces' terraform-config.yml)"
        echo "::set-output name=production-config::$(yq -o json -I 0 '.profiles.production.workspaces' terraform-config.yml)"

  terraform-apply-staging:
    name: "Terraform"
    needs: read-terraform-config
    uses: rewindio/github-action-terraform/.github/workflows/apply.yml@v1
    with:
      config: ${{ needs.read-terraform-config.outputs.staging_config }}
      profile: staging
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.MY_AWS_ACCESS_KEY_ID_STAGING }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.MY_AWS_SECRET_ACCESS_KEY_STAGING }}
      GITHUB_PAT: ${{ secrets.MY_GITHUB_PAT }}

  terraform-apply-production:
    name: "Terraform"
    needs: read-terraform-config
    uses: rewindio/github-action-terraform/.github/workflows/apply.yml@v1
    with:
      config: ${{ needs.read-terraform-config.outputs.production_config }}
      profile: production
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.MY_AWS_ACCESS_KEY_ID_STAGING }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.MY_AWS_SECRET_ACCESS_KEY_STAGING }}
      GITHUB_PAT: ${{ secrets.MY_GITHUB_PAT }}

```
