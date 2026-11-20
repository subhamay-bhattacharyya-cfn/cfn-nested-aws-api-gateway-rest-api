# CloudFormation Nested Stack Template : API Gateway Rest API

<!-- Row 1: Status - Most Important -->
[![Release](https://img.shields.io/github/v/release/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-rest-api?label=Release)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-rest-api/releases)&nbsp;[![Release Workflow](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-rest-api/actions/workflows/release.yaml/badge.svg)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-rest-api/actions/workflows/release.yaml)&nbsp;[![Issues](https://img.shields.io/github/issues/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-rest-api)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-rest-api/issues)&nbsp;[![Last Commit](https://img.shields.io/github/last-commit/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-rest-api)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-rest-api/commits)

<!-- Row 2: Code Quality -->
[![Top Language](https://img.shields.io/badge/Languages-Python%20%7C%20YAML-blue)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-rest-api)&nbsp;[![Commits](https://img.shields.io/github/commit-activity/t/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-rest-api)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-rest-api/commits)

<!-- Row 3: Tech Stack -->
[![CloudFormation](https://img.shields.io/badge/CloudFormation-IaC-orange?logo=amazonaws&logoColor=white)](https://aws.amazon.com/cloudformation/)&nbsp;[![Built with Claude Code](https://img.shields.io/badge/Built_with-Claude_Code-D97757?logo=anthropic&logoColor=white)](https://claude.ai/)

<!-- Row 4: Repository Info -->
[![Files](https://img.shields.io/github/directory-file-count/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-rest-api)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-rest-api)&nbsp;[![Repo Size](https://img.shields.io/github/repo-size/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-rest-api)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-rest-api)&nbsp;[![Release Date](https://img.shields.io/github/release-date/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-rest-api)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-rest-api/releases)

<!-- Row 5: Custom Metrics -->
[![Custom Endpoint](https://img.shields.io/endpoint?url=https://gist.githubusercontent.com/bsubhamay/8f0f81041e678b10907ee825e7be2307/raw/cfn-nested-aws-api-gateway-rest-api.json)](https://gist.github.com/subhamay-bhattacharyya-cfn/8f0f81041e678b10907ee825e7be2307)

---

This repository contains a nested CloudFormation template that creates an API Gateway REST API with a configurable name, description, and endpoint type.

## Overview

This is a **nested stack template** designed to be invoked from a parent/root CloudFormation stack. The template is stored in this repository and should be uploaded to an S3 bucket for reference by parent stacks.

## Template Files

- **`cloudformation/template.yaml`** — Nested template that creates the API Gateway REST API
- **`cloudformation/parameters.json`** — Parameter values used for deployment
- **`cloudformation/stack-config.json`** — Stack name and template/parameter file paths used by CI

## Template Features

- ✅ Regional, edge-optimized, or private endpoint types (selectable at deploy time)
- ✅ Configurable API name built from project, base name, account ID, environment, and region
- ✅ Configurable API description
- ✅ Optional CI suffix appended to the API name
- ✅ Tags for `ProjectName` and `Environment`

The REST API has no methods, resources, or deployment yet. Those are expected to be added by the parent stack or by a later change to this template.

## Parameters

| Parameter | Type | Default | Allowed Values / Constraints | Description |
| ----------- | ------ | --------- | ------------------------------- | ------------- |
| `ProjectName` | String | `cfn-nested` | 1–20 chars: lowercase letters, numbers, hyphens | Project name used as the API name prefix |
| `ApiBaseName` | String | `api` | 1–20 chars: lowercase letters, numbers, dots, hyphens | Base name for the API (account ID, environment, and region are appended) |
| `ApiDescription` | String | `API Description` | Max 1024 chars | Description of the REST API |
| `EndpointConfigurationType` | String | `REGIONAL` | `REGIONAL`, `EDGE`, `PRIVATE` | Endpoint type of the REST API |
| `Environment` | String | `devl` | `devl`, `test`, `prod` | Deployment environment |
| `CiSuffix` | String | `""` | Optional | Suffix appended to the API name when set (for example, a CI pipeline ID) |

### Endpoint Configuration Types

- **`REGIONAL`** — Serves clients in the same Region as the API.
- **`EDGE`** — Served through a CloudFront distribution managed by API Gateway.
- **`PRIVATE`** — Reachable only from within a VPC through an interface VPC endpoint. A resource policy is needed before clients can call it; this template does not create one.

## Outputs

- `RestApiId` — ID of the API Gateway REST API
- `RestApiRootResourceId` — Root resource ID of the REST API (used to add child resources)

## Usage

### 1. Upload the Template to S3

```bash
aws s3 cp cloudformation/template.yaml s3://your-cfn-bucket/templates/api-gateway-rest-api.yaml
```

### 2. Reference from a Parent Stack

In your parent/root CloudFormation template:

```yaml
ApiGatewayNestedStack:
  Type: AWS::CloudFormation::Stack
  Properties:
    TemplateURL: https://s3.amazonaws.com/your-cfn-bucket/templates/api-gateway-rest-api.yaml
    Parameters:
      ProjectName: !Ref ProjectName
      ApiBaseName: api
      ApiDescription: "Orders API"
      EndpointConfigurationType: REGIONAL
      Environment: !Ref Environment
      CiSuffix: !Ref CiSuffix
    Tags:
      - Key: Environment
        Value: !Ref Environment

Outputs:
  RestApiId:
    Value: !GetAtt ApiGatewayNestedStack.Outputs.RestApiId
  RestApiRootResourceId:
    Value: !GetAtt ApiGatewayNestedStack.Outputs.RestApiRootResourceId
```

### 3. Deploy Directly with AWS CLI

Parameter values are read from `cloudformation/parameters.json`, which should contain one key per parameter above:

```json
{
  "ProjectName": "cfn-nested",
  "ApiBaseName": "api",
  "ApiDescription": "API Description",
  "EndpointConfigurationType": "REGIONAL",
  "Environment": "devl",
  "CiSuffix": ""
}
```

```bash
aws cloudformation create-stack \
  --stack-name cfn-api-gateway-dev \
  --template-body file://cloudformation/template.yaml \
  --parameters file://cloudformation/parameters.json

aws cloudformation wait stack-create-complete --stack-name cfn-api-gateway-dev
```

#### With a CI Suffix

```bash
aws cloudformation create-stack \
  --stack-name cfn-api-gateway-dev-ci \
  --template-body file://cloudformation/template.yaml \
  --parameters \
    ParameterKey=ProjectName,ParameterValue=myproject \
    ParameterKey=ApiBaseName,ParameterValue=api \
    ParameterKey=Environment,ParameterValue=devl \
    ParameterKey=CiSuffix,ParameterValue=$CI_PIPELINE_ID
```

## API Naming Convention

The API name is built from the parameters as follows:

**Without CI Suffix:**

```bash
{ProjectName}-{ApiBaseName}-{AccountId}-{Environment}-{Region}
```

Example: `myproject-api-123456789012-devl-us-east-1`

**With CI Suffix:**

```bash
{ProjectName}-{ApiBaseName}-{AccountId}-{Environment}-{Region}-{CiSuffix}
```

Example: `myproject-api-123456789012-devl-us-east-1-pipeline-12345`

## Best Practices Implemented

- ✅ Unique API names per project, account, environment, and region
- ✅ Optional CI suffix support for unique deployments
- ✅ Tagging with `ProjectName` and `Environment`
- ✅ Parameter constraints enforced at deploy time (allowed values and patterns)

## License

MIT
