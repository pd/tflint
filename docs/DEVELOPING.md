# TFLint Developer Guide

The goal of this guide is to quickly understand how TFLint works and to help more developers contribute.

## Core Concept

TFLint is just a thin wrapper of Terraform. Configuration loading and expression evaluation etc. depend on Terraform's internal API, and it only provides an interface to do them as linter.

There are three important packages to understand its behavior:

- `tflint`
  - This package is the core of TFLint as a wrapper for Terraform. It allows accesses to `terraform/configs.Config` and `terraform/terraform.BuiltinEvalContext` and so on.
- `rules`
  - This package is a provider of all rules.
- `cmd`
  - This package is the entrypoint of the app.

## How does it work

These processes are described in [`cmd/cli.go`](https://github.com/terraform-linters/tflint/blob/master/cmd/cli.go).

### 1. Loading configurations

All Terraform's configuration files are represented as `configs.Config`. [`tflint/tflint.Loader`](https://github.com/terraform-linters/tflint/blob/master/tflint/loader.go) uses the `(*configs.Parser) LoadConfigDir` and `configs.BuildConfig` to access to `configs.Config` in the same way as Terraform.

Similarly, prepare `terraform.InputValues` using `(*configs.Parser) LoadValuesFile`.

### 2. Setting up a new Runner

A [`tflint/tflint.Runner`](https://github.com/terraform-linters/tflint/blob/master/tflint/runner.go) is initialized for each `configs.Config`. These have their own evaluation context for that module, represented as `terraform.BuiltinEvalContext`.

it uses `(*terraform.BuiltinEvalContext) EvaluateExpr` to evaluate expressions. Unlike Terraform, it provides a mechanism to determine if an expression can be evaluated.

### 3. Inspecting configurations

It inspects `configs.Config` via `tflint.Runner`. All rules implement the `Check` method that takes `tflint.Runner` as an argument, and emits an issue if needed.

## Building

You need Go 1.14 or later to build.

```console
$ make tools
$ make build
```

## Adding a new rule

You can use the rule generator to add new rules (Currently, this generator supports only AWS rules).

```console
$ go run ./rules/awsrules/generator
Rule name? (e.g. aws_instance_invalid_type): aws_instance_example
Create: rules/awsrules/aws_instance_example.go
Create: rules/awsrules/aws_instance_example_test.go
```

A template of rules and tests is generated. In order to inspect configuration files, you need to understand [the Runner API](https://github.com/terraform-linters/tflint/blob/master/tflint/runner.go).

Finally, don't forget to register the created rule with [the provider](https://github.com/terraform-linters/tflint/blob/master/rules/provider.go). After that the rule you created is enabled in TFLint.

## Editing the existing rules

Some rules are manually maintained, others are automatically generated. You should not edit directly anything that has been generated automatically.

### Manually maintained

Everything except the ones mentioned later is manually maintained. These can be changed directly.

### SDK-based

[SDK-based rules](https://github.com/terraform-linters/tflint/tree/master/rules/awsrules/models) are automatically generated from [aws-sdk](https://github.com/aws/aws-sdk-go) model definitions. For example, the valid regions of the S3 bucket are defined [here](https://github.com/aws/aws-sdk-go/blob/v1.23.11/models/apis/s3/2006-03-01/api-2.json#L1090-L1105). Based on this, we can generate a rule that checks whether a valid value is selected.

These definitions are linked to Terraform resource arguments by [mapping files](https://github.com/terraform-linters/tflint/tree/master/rules/awsrules/models/mappings). If you update these rules, you should update this mapping instead. The following is an example of a mapping file:

```hcl
import = "aws-sdk-go/models/apis/ec2/2016-11-15/api-2.json"

mapping "aws_instance" {
  instance_type = InstanceType
}
```

The left is a Terraform argument, and the right is the shape name of the aws-sdk model definition. From this mapping, a rule is automatically generated by the following command:

```console
go generate ./rules/awsrules/model
```

### API-based

[API-based rules](https://github.com/terraform-linters/tflint/tree/master/rules/awsrules/api) are automatically generated from [definition files](https://github.com/terraform-linters/tflint/tree/master/rules/awsrules/api/definitions). An example definition file is shown below:

```hcl
rule "aws_instance_invalid_iam_profile" {
    resource      = "aws_instance"
    attribute     = "iam_instance_profile"
    source_action = "ListInstanceProfiles"
    template      = "\"%s\" is invalid IAM profile name."
}
```

In this definition file, a rule that checks whether the `aws_instance.*.iam_instance_profile` value is included in the return values of `source_action` is generated. The rule is generated with `go generate`:

```console
go generate rules/awsrules/api
```

## Committing

Before commit, please install [pre-commit](https://pre-commit.com/) and install pre-commit hooks by running `pre-commit install` in root directory of your local checkout.
