# Terraform Guidelines

# Introduction

This document outlines best practices for writing Terraform modules and using them. It recommends standard structure, variable usage and naming conventions and output formats. It also suggests using the `terraform-null-label` module to ensure consistent resource naming, small opinionated modules, and composability of modules. These guidelines help ensure that the code is secure and maintainable.

# Prerequisites

## ASDF

We use asdf for managing the runtime versions of our required tools. You can read more about it [here](https://asdf-vm.com/).

Every module does have a `.tool-versions` file where every tool version is defined.

Please install and configure asdf beforehand. Follow the guide [here](https://asdf-vm.com/guide/getting-started.html).

**Example:**

```plaintext
terraform 1.3.7
terraform-docs 0.16.0
tflint 0.43.0
pre-commit 3.0.1
```

**Installation:**

`asdf install`

This will install all the defined tool versions listed in the file.

# Structure

We enforce the standard module structure that is recommended from Hashicorp you can read more about it [here](https://developer.hashicorp.com/terraform/language/modules/develop#standard-module-structure).

Configurations like variables, outputs and resources should be sorted after logic and not a-z.

## Context

### File

Every module should have a `context.tf` file to generate consistent names and tags for resources. You can read more about it [here](https://github.com/cloudposse/terraform-null-label).

### `label_order`

Additionally, we want to be able to customize the `id` with the `label_order` for different use cases. Changing only the label order has the benefit of keeping `tags` at all times nevertheless how the `id` might look in the end.

These should be configurable based on the AWS Service where the label module gets used. For example ec2, ddb, ecs, eks…

Here is an example of how every label module call should be extended with a variable to modify the `label_order`:

```typescript
module "example_label" {
  source  = "cloudposse/label/null"
  version = "0.25.0"

  label_order = var.label_orders.ec2

  context = module.this.context
}
```

As a variable, we want to have `label_orders` with every object optional to configure as seen below:

```typescript
variable "label_orders" {
  type = object({
    ec2 = optional(list(string))
  })
default     = {}
  description = "Overrides the `labels_order` for the different labels to modify ID elements appear in the `id`"
}
```

That variable will get expanded and passed in every module that uses extra label modules.

## Use proper datatype

## Use minimum version pinning on all providers

## Use `locals`

Using `locals` makes code more descriptive and maintainable. Rather than using complex expressions as parameters to some terraform resource, instead move that expression to a `local` and reference the `local` in the resource.

# Variables

## Use upstream module or provider variable names where applicable

When writing a module that accepts `variable` inputs, make sure to use the same names as the upstream to avoid confusion and ambiguity.

## Use all lower-case with underscores as separators

Avoid introducing any other syntaxes commonly found in other languages such as CamelCase or pascalCase. For consistency we want all variables to look uniform. This is also inline with the [HashiCorp naming conventions](https://www.terraform.io/docs/extend/best-practices/naming.html).

## Use positive variable names to avoid double negatives

All `variable` inputs that enable/disable a setting should be formatted `...._enabled` (e.g. `encryption_enabled`). It is acceptable for default values to be either `false` or `true`.

## Use feature flags to enable/disable functionality

All modules should incorporate feature flags to enable or disable functionality. All feature flags should end in `_enabled` and should be of type `bool`.

## Use description field for all inputs

All `variable` inputs need a `description` field. When the field is provided by an upstream provider (e.g. `terraform-aws-provider`), use same wording as the upstream docs.

## Use sane defaults where applicable

Modules should be as turnkey as possible. The `default` value should ensure the most secure configuration (E.g. with encryption enabled).

## Use variables for all secrets with no `default` value

All `variable` inputs for secrets must never define a `default` value. This ensures that `terraform` is able to validate user input. The exception to this is if the secret is optional and will be generated for the user automatically when left `null` or `""` (empty).

# Outputs

## Use description field for all outputs

All outputs must have a `description` set. The `description` should be based on (or adapted from) the upstream terraform provider where applicable. Avoid simply repeating the variable name as the output `description`.

## Use well-formatted snake case output names

Avoid introducing any other syntaxes commonly found in other languages such as CamelCase or pascalCase. For consistency we want all variables to look uniform. It also makes code more consistent when using outputs together with terraform [`remote_state`](https://www.terraform.io/docs/providers/terraform/d/remote_state.html) to access those settings from across modules.

## Use symmetrical names

We prefer to keep terraform outputs symmetrical as much as possible with the upstream resource or module, with exception of prefixes. This reduces the amount of entropy in the code or possible ambiguity, while increasing consistency. Below is an example of what **not* to do. The expected output name is `user_secret_access_key`. This is because the other IAM user outputs in the upstream module are prefixed with `user_`, and then we should borrow the upstream’s output name of `secret_access_key` to become `user_secret_access_key` for consistency.

# Naming Conventions

## Use a programmatically consistent naming convention

All resource names (E.g. things provisioned on AWS) must follow a consistent convention. The reason this is so important is that modules are frequently composed inside of other modules. Enforcing consistency increases the likelihood that modules can invoke other modules without colliding on resource names.

To enforce consistency, we require that all modules use the [`terraform-null-label`](https://github.com/cloudposse/terraform-null-label) module. With this module, users have the ability to change the way resource names are generated such as by changing the order of parameters or the delimiter. While the module is opinionated on the parameters, it’s proved invaluable as a mechanism for generating consistent resource names.

# Module Design

## Small Opinionated Modules

We believe that modules should do one thing very well. But in order to do that, it requires being opinionated on the design. Simply wrapping terraform resources for the purposes of modularising code is not that helpful. Implementing a specific use-case of those resource is more helpful.

## Composable Modules

Write all modules to be easily composable into other modules. This is how we’re able to achieve economies of scale and stop re-inventing the same patterns over and over again.

## Use `variable` inputs

Modules should accept as many parameters as possible. Avoid using inputs of `type = object` since they are harder to document. Of course, this is not a hard rule and sometimes objects just make the most sense. Just be weary of the ability for tools like [`terraform-docs`](https://github.com/segmentio/terraform-docs) to be able to generate meaningful documentation.

# Module Usage

## Use Terraform registry format with exact version numbers

There are many ways to express a module’s source. Our convention is to use Terraform registry syntax with an explicit version.

```hcl
source  = "justtrack/ecs-app/aws"
version = "1.2.0"
```

The reason to pin to an explicit version rather than a range like `>= 1.2.0` is that any update is capable of breaking something. Any changes to your infrastructure should be implemented and reviewed under your control, not blindly automatic based on when you deployed it.

## Utilizing public modules

Please use public modules everywhere possible. That helps the overall maintenance needed and time spend during development. We recommend using public modules from these organizations as they are pretty well maintained and follow terraform module best practices.

1. [terraform-aws-modules](https://github.com/terraform-aws-modules)
2. [cloudposse](https://github.com/cloudposse)

Please utilize them in the order given above if possible.

