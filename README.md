# terraform-aws-lambda-at-edge-cognito-authentication
This terraform module creates a Lambda@Edge Lambda to be used within a viewer policy of a CloudFront Distribution to
enforce Cognito Authentication through a configured Cognito User Pool.

## How it Works 
![Sequence Diagram](docs/sequence.png)

## Requirements
- Terraform version >= 1.0.X 
- NodeJS + NPM (compatible with NodeJS 20.X.X)
  - Used for `npm ci` dependency installation for Lambda@Edge Bundle.
- Terraform AWS Provider in `us-east-1`
  - Requirement for CloudFront + Lambda@Edge runtime.
- Existing Cognito User Pool and User Pool Client.

## Usage
To use this module, source it into your Terraform project like so:

```
data "aws_cognito_user_pools" "user_pool" {
  name     = "my_cognito_pool_name"
}

resource "aws_cognito_user_pool_client" "cognito_client" {
  ...
}

module "cloudfront_cognito_auth_lambda" {
  source    = "git::https://github.com/disney/terraform-aws-lambda-at-edge-cognito-authentication.git?ref=<version-ref>"
  name                                      = "my-application"
  cognito_user_pool_name                    = data.aws_cognito_user_pools.user_pool.name
  cognito_user_pool_region                  = "us-east-1"
  cognito_user_pool_id                      = aws_cognito_user_pool_client.cognito_client.user_pool_id
  cognito_user_pool_app_client_id           = aws_cognito_user_pool_client.cognito_client.id

  tags = { foo = "bar" }
}
```

You can use the output of this module to pass as a rule on your CloudFront distribution cache on viewer-requests (typically on an S3 Bucket Origin with an OAI).

```
resource "aws_cloudfront_distribution" "my_cloudfront_distribution" {
  ...

  ordered_cache_behavior {
    ...
    lambda_function_association {
      event_type   = "viewer-request"
      lambda_arn   = module.cloudfront_cognito_auth_lambda.qualified_arn
    }
  }
}

```
### Lambda Configuration Mode

The module supports three different strategies to make the required cognito-at-edge config available to the lambda function, controlled by the `lambda_config_mode` variable.

> [!NOTE] 
> While the default config mode is `dynamic` (for backwards compatibility), either of the other modes is a better choice for most users.

#### `dynamic`

In this mode, the lambda introspects the IAM role/policy to determine the SSM Parameter that contains the config.  This is the current/default behaviour if not supplied by the calling module.

This requires at least three round trips to AWS, which can cause 503 responses from Cloudfront during lambda cold starts when the round trips and cognito-at-edge initialisation collectively takes longer than the maximum lambda-at-edge run time (5 seconds). The lambda code does not need to be redeployed when config changes are made (so terraform applies are fast).

#### `hybrid`

In this mode, the name of the parameter store entry that contains the config is written into a config file shipped with the lambda. Versus 'dynamic' mode this removes the need for IAM introspection, which saves two round trips to AWS, which reduces the likelihood that cold starts exceed the max run time. The net result is that this mode will produce signicantly less 503's, particularly if your cloudfront distribution is used in regions a long distance from us-east-1, or only used sporadically. Changes to the config of the lambda will still be fast, but changing the actual SSM entry that holds the config will force a lambda redeploy (which takes minutes). This mode will be cheaper to run than 'dynamic' (as on average cold starts consume 1-2 seconds less lambda runtime).

#### `static`

In this mode, the entire cognito-at-edge config is written into a config file shipped with the lambda.  Versus 'hybrid' mode, this saves another round trip to AWS (the fetching of the config from SSM), which yields even faster cold start times and will therefore be cheaper still to run.  The downsides:
* any config change requires a reprovisioning of the lambda (slow)
* secrets passed into the module config will be written to the config file in the local file system, which may lead to secret leakage.   
To prevent this happening unintentionally, if the calling module sets `cognito_user_pool_app_client_secret` and `lambda_config_mode = static`, then plan/apply will fail unless `lambda_config_allow_insecure_secret_storage = true` is also set.

### Cloudwatch logging
If a Lambda@Edge function has IAM permission to `logs:CreateLogGroup` then it will create a Cloudwatch log group called `/aws/lambda/us-east-1.<lambda-name>` in every CloudFront region that serves a request, which can result in logging proliferating across many regions, as described in https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/edge-functions-logs.html#lambda-at-edge-logs .  This auto-created log group has no retention policy set, so in addition to Cloudwatch ingest costs, Cloudwatch storage costs increase over time for the life of the function.  Setting `cloudwatch_enable_log_group_create = false` will mean that the IAM policies in the module will not grant `logs:CreateLogGroup` to the edge-auth Lambda@Edge function, and if the log group does not exit, no logging happens (avoiding Cloudwatch ingest and storage costs).  Note:
- If the variable is `false` and logging is required in a specific region, the caller is responsible for ensuring the log group exists in that region.
- Retroactively setting the variable to `false` is not sufficient to disable logging if the log group already exists in a region - manual deletion of the unwanted log groups will be required to prevent further logging.
- If a policy outside the module grants the edge-auth function rights to `logs:CreateLogGroup`, then Lambda@Edge will still create the log group and log.

### Lambda@Edge Destroy Issue
Note that if a destroy action is performed on this terraform module, terraform is unable to delete the Lambda@Edge that was published as a part of this infrastructure (This is noted by this [issue](https://github.com/hashicorp/terraform-provider-aws/issues/1721) on the Terraform provider). It will only be removed from the terraform state as the `skip_destroy` flag is set to true.

In order to properly delete this resource, it should be manually cleaned up, [instructions here](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/lambda-edge-delete-replicas.html).


<!-- BEGINNING OF PRE-COMMIT-TERRAFORM DOCS HOOK -->
## Requirements

| Name | Version |
|------|---------|
| <a name="requirement_terraform"></a> [terraform](#requirement\_terraform) | ~> 1.0 |
| <a name="requirement_aws"></a> [aws](#requirement\_aws) | >= 4.57 |

## Providers

| Name | Version |
|------|---------|
| <a name="provider_archive"></a> [archive](#provider\_archive) | 2.4.0 |
| <a name="provider_aws"></a> [aws](#provider\_aws) | 5.26.0 |
| <a name="provider_local"></a> [local](#provider\_local) | 2.5.2 |
| <a name="provider_null"></a> [null](#provider\_null) | 3.2.2 |

## Modules

No modules.

## Resources

| Name | Type |
|------|------|
| [aws_iam_role.lambda_at_edge](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role) | resource |
| [aws_iam_role_policy.allow_cloudwatch_logs](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role_policy) | resource |
| [aws_iam_role_policy.lambda_edge_self_role_read](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role_policy) | resource |
| [aws_iam_role_policy.ssm_parameter_decrypt](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role_policy) | resource |
| [aws_iam_role_policy.ssm_parameter_permission_for_lambda_auth](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role_policy) | resource |
| [aws_kms_key.ssm_kms_key](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/kms_key) | resource |
| [aws_lambda_function.cloudfront_auth_edge](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lambda_function) | resource |
| [aws_ssm_parameter.lambda_configuration_parameters](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/ssm_parameter) | resource |
| [local_file.lambda_configuration](https://registry.terraform.io/providers/hashicorp/local/latest/docs/resources/file) | resource |
| [null_resource.install_lambda_dependencies](https://registry.terraform.io/providers/hashicorp/null/latest/docs/resources/resource) | resource |
| [archive_file.lambda_edge_bundle](https://registry.terraform.io/providers/hashicorp/archive/latest/docs/data-sources/file) | data source |
| [aws_caller_identity.current](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/caller_identity) | data source |
| [aws_iam_policy_document.allow_cloudwatch_logs](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/iam_policy_document) | data source |
| [aws_iam_policy_document.allow_lambda_edge_self_role_read](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/iam_policy_document) | data source |
| [aws_iam_policy_document.allow_lambda_service_assume](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/iam_policy_document) | data source |
| [aws_iam_policy_document.allow_ssm_parameter_decrypt](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/iam_policy_document) | data source |
| [aws_iam_policy_document.allow_ssm_parameter_permission_for_lambda_auth](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/iam_policy_document) | data source |

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| <a name="input_cloudwatch_enable_log_group_create"></a> [cloudwatch\_enable\_log\_group\_create](#input\_cloudwatch\_enable\_log\_group\_create) | Allow Lambda@Edge to create log groups in cloudwatch, defaults to true. | `bool` | `true` | no |
| <a name="input_cognito_additional_settings"></a> [cognito\_additional\_settings](#input\_cognito\_additional\_settings) | Map of any to configure any additional cognito@edge parameters not handled by this module. | `any` | `{}` | no |
| <a name="input_cognito_cookie_expiration_days"></a> [cognito\_cookie\_expiration\_days](#input\_cognito\_cookie\_expiration\_days) | Number of days to keep the cognito cookie valid. | `number` | `7` | no |
| <a name="input_cognito_disable_cookie_domain"></a> [cognito\_disable\_cookie\_domain](#input\_cognito\_disable\_cookie\_domain) | Sets domain attribute in cookies, defaults to false. | `bool` | `false` | no |
| <a name="input_cognito_log_level"></a> [cognito\_log\_level](#input\_cognito\_log\_level) | Logging level. Default: 'silent' | `string` | `"silent"` | no |
| <a name="input_cognito_redirect_path"></a> [cognito\_redirect\_path](#input\_cognito\_redirect\_path) | Optional path to redirect to after a successful cognito login. | `string` | `""` | no |
| <a name="input_cognito_user_pool_app_client_id"></a> [cognito\_user\_pool\_app\_client\_id](#input\_cognito\_user\_pool\_app\_client\_id) | Cognito User Pool App Client ID for the targeted user pool. | `string` | n/a | yes |
| <a name="input_cognito_user_pool_app_client_secret"></a> [cognito\_user\_pool\_app\_client\_secret](#input\_cognito\_user\_pool\_app\_client\_secret) | Cognito User Pool App Client Secret for the targeted user pool. NOTE: This is currently not compatible with AppSync applications. | `string` | `null` | no |
| <a name="input_cognito_user_pool_domain"></a> [cognito\_user\_pool\_domain](#input\_cognito\_user\_pool\_domain) | Optional: Full Domain of the Cognito User Pool to utilize. Mutually exclusive with 'cognito\_user\_pool\_name'. | `string` | `""` | no |
| <a name="input_cognito_user_pool_id"></a> [cognito\_user\_pool\_id](#input\_cognito\_user\_pool\_id) | Cognito User Pool ID for the targeted user pool. | `string` | n/a | yes |
| <a name="input_cognito_user_pool_name"></a> [cognito\_user\_pool\_name](#input\_cognito\_user\_pool\_name) | Name of the Cognito User Pool to utilize. Required if 'cognito\_user\_pool\_domain' is not set. | `string` | `""` | no |
| <a name="input_cognito_user_pool_region"></a> [cognito\_user\_pool\_region](#input\_cognito\_user\_pool\_region) | AWS region where the cognito user pool was created. | `string` | `"us-west-2"` | no |
| <a name="input_lambda_config_allow_insecure_secret_storage"></a> [lambda\_config\_allow\_insecure\_secret\_storage](#input\_lambda\_config\_allow\_insecure\_secret\_storage) | Allow secrets to be stored in the lambda config file, defaults to false. | `bool` | `false` | no |
| <a name="input_lambda_config_mode"></a> [lambda\_config\_mode](#input\_lambda\_config\_mode) | Which strategy to use to supply config to the lambda function, defaults to 'dynamic'. | `string` | `"dynamic"` | no |
| <a name="input_lambda_runtime"></a> [lambda\_runtime](#input\_lambda\_runtime) | Lambda runtime to utilize for Lambda@Edge. | `string` | `"nodejs20.x"` | no |
| <a name="input_lambda_timeout"></a> [lambda\_timeout](#input\_lambda\_timeout) | Amount of timeout in seconds to set on for Lambda@Edge. | `number` | `5` | no |
| <a name="input_name"></a> [name](#input\_name) | Name to prefix on all infrastructure created by this module. | `string` | n/a | yes |
| <a name="input_tags"></a> [tags](#input\_tags) | Map of tags to attach to all AWS resources created by this module. | `map(string)` | `{}` | no |

## Outputs

| Name | Description |
|------|-------------|
| <a name="output_arn"></a> [arn](#output\_arn) | ARN for the Lambda@Edge created by this module. |
| <a name="output_function_name"></a> [function\_name](#output\_function\_name) | Name of the Lambda@Edge created by this module. |
| <a name="output_qualified_arn"></a> [qualified\_arn](#output\_qualified\_arn) | Qualified ARN for the Lambda@Edge created by this module. |
<!-- END OF PRE-COMMIT-TERRAFORM DOCS HOOK -->
