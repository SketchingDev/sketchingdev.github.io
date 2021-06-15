---
layout: post
title:  "Deploying AppSync with Terraform"
date:   2019-03-14 00:00:00
categories: terraform appsync aws
image-base: /assets/images/posts/2019-03-14-deploying-appsync-with-terraform
---

On the 28th November ’17 AWS introduced the world to [AppSync](https://aws.amazon.com/appsync/), its new fully managed GraphQL service that bridges the gap between your front-end and data-ladened backend. Most notably its features include data synchronisation (no more polling!) and offline capabilities.

However long after its launch Terraform’s AWS Provider still doesn’t have all the resources necessary to deploy a complete instance of AppSync ([although they are very close to being released](https://github.com/terraform-providers/terraform-provider-aws/pull/6451)). Leaving some developers turning to  [Serverless](https://github.com/sid88in/serverless-appsync-plugin), manually intervening after deployments or using external scripts to make up for this short-fall.

![Diagram of AppSync components]({{ page.image-base }}/appsync-diagram.png)

The following shows how use Terraform’s [cloudformation_stack resource](https://www.terraform.io/docs/providers/aws/r/cloudformation_stack.html) to create the missing resources - removing the need for the aforementioned workarounds and making future migration a doddle.

If you’d prefer to jump into a complete demonstration then head over to [GitHub - SketchingDev/terraform-appsync-demo](https://github.com/SketchingDev/terraform-appsync-demo).

## What AppSync resources are supported
Currently the AWS Provider supports deploying the API, an authentication key and data sources, but if you actually want to resolve a request you’re going to need a Schema and Resolver, which are both missing.

* GraphQL API
* API Key
* DataSource
* ~~Schema~~
* ~~Resolvers~~

```terraform
resource "aws_appsync_graphql_api" "people" {
  name                = "${var.appsync_name}"
  authentication_type = "API_KEY"
}

resource "aws_appsync_api_key" "people_api" {
  api_id  = "${aws_appsync_graphql_api.people.id}"
}

resource "aws_appsync_datasource" "people" {
  api_id           = "${aws_appsync_graphql_api.people.id}"
  name             = "${var.datasource_name}"
  service_role_arn = "${aws_iam_role.api.arn}"
  type             = "AMAZON_DYNAMODB"

  dynamodb_config {
    table_name = "${aws_dynamodb_table.people.name}"
  }
}
```

## CloudFormation to the rescue
Luckily Terraform’s AWS Provider offers us a lifeline for dealing with missing resources thanks to its [aws_cloudformation_stack](https://www.terraform.io/docs/providers/aws/r/cloudformation_stack.html) resource - allowing us to harness CloudFormation templates within Terraform.

In case you’ve not come across [CloudFormation](https://aws.amazon.com/cloudformation/)  before it is AWS’s native infrastructure-as-code offering that - just like Terraform - allows you to automate the provisioning of AWS services via ‘templates’. Importantly though since it is owned by AWS it supports most (if not all) of their services.

### Downsides
Unfortunately though little in life comes without a price and for us it’s that Terraform hands responsibility of creation, updating and destruction to CloudFormation with some unfortunate consequences:

* Resource recreation due to minor changes - Updating the CloudFormation templates or its variables will likely lead to the resource being recreated
* Manual intervention for some failures - It’s often a result of human error but if there is an error creating a CloudFormation resource - such as providing an invalid template value - then subsequent deploys might fail with a `ROLLBACK_COMPLETE` message. This is [resolved by manually destroying the resource](https://www.quora.com/Why-can-CloudFormation-stacks-in-a-ROLLBACK_COMPLETE-state-no-longer-be-updated-and-can-only-be-deleted)
*  No dependency inference - Terraform won’t be able to infer dependencies so you’ll likely depend on [`depends_on`](https://www.terraform.io/docs/configuration/resources.html#depends_on-hidden-resource-dependencies)

### Terraforming the missing pieces
The [aws_cloudformation_stack](https://www.terraform.io/docs/providers/aws/r/cloudformation_stack.html) resource has two main arguments, the CloudFormation template and its parameters. In the example below the templates ([which can be seen here](https://github.com/SketchingDev/terraform-appsync-demo/tree/master/cloudformation-templates)) define AppSync’s schema and resolver and are stored alongside the modules files. They can then be referenced using the [local_file data source](https://www.terraform.io/docs/providers/local/d/file.html), reducing duplication between resolvers.

It’s then just a case of providing the parameters and any hidden dependencies via the `depends_on` meta-argument.

#### AppSync Schema

```terraform
data "local_file" "cloudformation_schema_template" {
  filename = "${path.module}/cloudformation-templates/schema.json"
}

data "local_file" "schema" {
  filename = "${path.module}/people-api/schema.graphql"
}

resource "aws_cloudformation_stack" "api_schema" {
  depends_on = ["aws_appsync_datasource.people"]
  name = "${var.appsync_name}-schema"

  parameters = {
    graphQlApiId = "${aws_appsync_graphql_api.people.id}"
    graphQlSchema = "${data.local_file.schema.content}"
  }

  template_body = "${data.local_file.cloudformation_schema_template.content}"
}
```


#### AppSync Resolver

```terraform
data "local_file" "create_source_request_mapping" {
  filename = "${path.module}/people-api/resolvers/createPerson-request-mapping-template.txt"
}

data "local_file" "create_source_response_mapping" {
  filename = "${path.module}/people-api/resolvers/createPerson-response-mapping-template.txt"
}

resource "aws_cloudformation_stack" "create_person_resolver" {
  depends_on = [
    "aws_appsync_datasource.people",
    "aws_cloudformation_stack.api_schema"
  ]
  name = "${var.appsync_name}-create-person-resolver"

  parameters = {
    graphQlApiId = "${aws_appsync_graphql_api.people.id}"
    dataSourceName = "${var.datasource_name}"
    fieldName = "createPerson"
    typeName = "Mutation"
    requestMappingTemplate = "${data.local_file.create_source_request_mapping.content}"
    responseMappingTemplate = "${data.local_file.create_source_response_mapping.content}"
  }

  template_body = "${data.local_file.cloudformation_resolver_template.content}"
}
```


## Deploying your API
You can now deploy your API with the usual `terraform apply` command:

```shell
$ terraform apply -auto-approve

...

Apply complete! Resources: 9 added, 0 changed, 0 destroyed.

Outputs:

api_key = <sensitive>
api_uris = {
  GRAPHQL = https://s3tfchjklnfz7d76rqqqfvjf7m.appsync-api.us-east-1.amazonaws.com/graphql
}
```

Check out a complete example of this at [GitHub - SketchingDev/terraform-appsync-demo](https://github.com/SketchingDev/terraform-appsync-demo).
