# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a **CloudFormation template repository** that provides a reusable nested stack template for creating an API Gateway REST API. The template follows the nested stack pattern and is designed to be referenced by parent CloudFormation stacks.

**Key characteristics:**

- Nested CloudFormation template (referenced via `TemplateURL`)
- Configurable API name built from project, base name, account ID, environment, and region
- Configurable API description and endpoint type (`REGIONAL`, `EDGE`, `PRIVATE`)
- Optional CI suffix appended to the API name
- Automated semantic versioning and releases
- AWS OIDC authentication for CI/CD deployments

## Project Structure

```text
cloudformation/
├── template.yaml                 # Nested template: API Gateway REST API
├── parameters.json               # Parameter values for deployment
└── stack-config.json             # Stack name and template/parameter file paths used by CI

.github/workflows/
├── ci.yaml                       # Loads stack config and runs the reusable CI build
├── release.yaml                  # Semantic release on push to main
├── create-branch.yaml            # Auto-create feature branches from issues
├── claude.yaml                   # Claude Code workflow
├── claude-code-review.yaml       # Claude Code review workflow
├── notify.yaml                   # Notification workflow
└── setup-environments.yaml       # GitHub environment setup

scripts/plugins/
├── release.config.js             # Semantic-release configuration
└── (other release plugins)       # Custom commit analysis, notes generation

.claude/
├── .skills/                      # Project skills (README, package.json, CI, contribution)
├── settings.json                 # Claude Code workspace settings
└── settings.local.json           # Local overrides

.devcontainer/
└── devcontainer.json             # Dev container setup (Node.js 20)

.env/
└── environments.yaml             # Environment definitions

package.json                      # Dependencies: semantic-release, commitizen
README.md                         # Template documentation and usage examples
```

## Development Commands

### Install dependencies

```bash
npm ci
```

### Trigger semantic release (usually automatic on main)

```bash
npm run release
```

### Commit with conventional commit format

```bash
npx cz commit
```

Select `feat`, `fix`, or `chore` type. Only `feat` and `fix` trigger releases.

## Key Architecture Concepts

### Nested Stack Pattern

This repo provides a **nested stack template** — a template that is referenced from a parent/root CloudFormation stack via `TemplateURL`.

- **Parent stack** calls: `AWS::CloudFormation::Stack` with `TemplateURL` pointing to S3
- **Nested template** exposes values through its `Outputs` section
- Parent retrieves outputs via `!GetAtt NestedStack.Outputs.OutputKey`

### API Naming Convention

The REST API name is built from parameters:

```bash
{ProjectName}-{ApiBaseName}-{AccountId}-{Environment}-{Region}[-{CiSuffix}]
```

Example: `myproject-api-123456789012-devl-us-east-1`

This is implemented with a `HasCiSuffix` condition selecting between the two `!Sub` forms in `RestApi.Properties.Name`.

This ensures:

- Uniqueness across AWS accounts and regions
- Environment isolation
- Consistent naming for infrastructure automation

### Parameter-Driven Configuration

The template's parameters are grouped in `Metadata` under `AWS::CloudFormation::Interface`:

- **Common Bucket Configuration**: `ProjectName`, `Environment` (the group label is a leftover from an earlier S3 template)
- **API Configuration**: `ApiBaseName`, `ApiDescription`, `EndpointConfigurationType`
- **Advanced Configuration**: `CiSuffix`

Every parameter listed in `Metadata` must also be defined in `Parameters`.

## Key Files to Understand

### `cloudformation/template.yaml`

**Purpose:** Creates an API Gateway REST API (`AWS::ApiGateway::RestApi`)

**Key inputs:**

- `ProjectName` (required, default: `cfn-nested`): Project prefix for the API name and `ProjectName` tag
- `ApiBaseName` (default: `api`): Base name component of the API name
- `ApiDescription` (default: `API Description`): Description of the REST API
- `EndpointConfigurationType` (default: `REGIONAL`; allowed: `REGIONAL`, `EDGE`, `PRIVATE`): Endpoint type
- `Environment` (default: `devl`; allowed: `devl`, `test`, `prod`): Environment label, also used as the `Environment` tag
- `CiSuffix` (default: empty): Optional suffix appended to the API name

**Key outputs:**

- `RestApiId`: REST API ID
- `RestApiRootResourceId`: Root resource ID, used to add child resources

**Features:**

- Conditional naming: different API name with/without CI suffix
- Tags: `ProjectName` and `Environment`
- The REST API has no methods, resources, or deployment yet

### `cloudformation/parameters.json`

Parameter values passed to the stack. Keys must match parameters defined in `template.yaml`, because CloudFormation rejects unknown keys.

### `cloudformation/stack-config.json`

Read by CI to get the stack name and the template and parameter file paths (`templateFile`, `parameterFile`).

### `.github/workflows/ci.yaml`

**Triggered on:**

- Manual `workflow_dispatch`
- Pushes to `feature/**` and `bug/**` branches
- Pull requests targeting `main`

**Path filter:** Only runs if changes to `cloudformation/**` or `.github/workflows/ci.yaml`

**Phases:**

1. **Load Stack Configuration:** Reads `cloudformation/stack-config.json` into job outputs
2. **CI Build:** Calls the reusable workflow `subhamay-bhattacharyya-gha/cfn-ci-build-reusable-wf` with the `ci` environment, stack name, template file, and parameter file
3. **Generate Changelog:** On `main` only, runs `git-cliff` and commits `CHANGELOG.md`
4. **Create Release:** On `main` only, runs `subhamay-bhattacharyya-gha/create-release-action`

## Testing & Validation

**Manual template validation:**

```bash
aws cloudformation validate-template --template-body file://cloudformation/template.yaml
```

**Manual stack deployment:**

```bash
aws cloudformation create-stack \
  --stack-name cfn-api-gateway-dev \
  --template-body file://cloudformation/template.yaml \
  --parameters file://cloudformation/parameters.json \
  --region us-east-1
```

The CI workflow runs the build phase automatically on push and pull request.

## AWS Credentials & Environment Variables

**GitHub environment `ci`:** The CI build uses the `ci` environment for OIDC-based AWS access. The GitHub OIDC provider must trust the role used by the reusable workflow.

## Conventional Commits & Release Flow

This repo enforces conventional commits to drive semantic versioning:

```bash
npx cz commit
```

Commit types:

- `feat: add support for X` → triggers MINOR release
- `fix: correct behavior of Y` → triggers PATCH release
- `chore: update deps` → no release
- `docs: clarify README` → no release

Only commits to `main` trigger releases. Feature branches use this format but releases happen on merge to main.

## When Modifying Templates

1. **Edit the template YAML** at `cloudformation/template.yaml`
2. **Update `cloudformation/parameters.json`** when parameters are added, removed, or renamed
3. **Update the `Metadata` parameter groups and labels** to match `Parameters`
4. **Test locally** with `aws cloudformation validate-template`
5. **Create a PR** with a conventional commit message (e.g., `feat: add endpoint type parameter`)
6. **CI validates and builds** automatically
7. **Merge to main** → changelog and release jobs run

## Dev Container

Pre-configured with:

- Node.js 20
- GitHub Copilot extension

Use via VS Code: `code --remote-container-url <repo-url>`

## Current Branch

Main branch is the release branch. Feature work branches from here and merges back via PR. Branch naming follows: `{type}/CFN-{issue-number}-{slug}` (e.g., `feature/CFN-42-add-encryption`).
