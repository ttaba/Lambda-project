# Lambda-project
//setup for terraform / aws packages
terraform {
  required_version = ">= 0.12" //any version great than or = to 0.12

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 3.61"
    }
  }
}

//set region where work will be done
provider "aws" {
  region = var.region
}

//zipping up source code files that are within directory that terraform is working out of
data "archive_file" "myzip" {
  type        = "zip"
  source_file = "${path.module}/index.js" 
  output_path = "${path.module}/SalesManager_GetSettings.zip"
}

//set role to allow dev account the access to create roles in sandbox
resource "aws_iam_role_policy_attachment" "lambda_basic_execution_role" {
  role       = aws_iam_role.SalesManager_lambda_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

//creating the lambda and housing files within lambda
resource "aws_lambda_function" "create_lambda" {
  filename      = "${data.archive_file.myzip.output_path}"
  function_name = var.LambdaName  //Name of Lambda
  role          = aws_iam_role.SalesManager_lambda_role.arn
  handler       = var.lambda_handler_name//NodeJS script
  runtime       = var.runtime      //each Lambda can point to its own runtime
  layers        = ["arn:aws:lambda:${var.region}:${data.aws_caller_identity.current.account_id}:layer:Datadog-<Node12-x>:<66>","arn:aws:lambda:${var.region}:${data.aws_caller_identity.current.account_id}:layer:Datadog-Extension:<15>",]



environment {
    variables = {
      AWS_LAMBDA_EXEC_WRAPPER = "/opt/dynatrace" # Use the wrapper from the layer
      DT_TENANT = "dmj55461"
      DT_CLUSTER_ID = "-1123358233"
      DT_CONNECTION_BASE_URL = "https://dmj55461.live.dynatrace.com"
      DT_CONNECTION_AUTH_TOKEN = "dt0a01.dmj55461.7a1a389f7928f4843435137641d442719613e6de807a151229cef0bbdc9b3e4a"
    }
  }
}

//creating the role policy for the lambda
resource "aws_iam_role" "SalesManager_lambda_role" {
name = var.LambdaRoleName

assume_role_policy = <<EOF
{
      "Version": "2012-10-17",
      "Statement":[
            {
              "Action": "sts:AssumeRole",
              "Principal": {
              "Service": "lambda.amazonaws.com"
              },
              "Effect": "Allow",
              "Sid": ""
            }
        ]
    }
    EOF
}

//naming SNS 
resource "aws_sns_topic" "default"{
name = var.SnsName
}

//sns setup for lambda
resource "aws_lambda_permission" "sns_usage" {
  statement_id = "AllowExecutionFromSNS"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.create_lambda.function_name
  principal     = "sns.amazonaws.com"
  source_arn    = aws_sns_topic.default.arn
}

//setup pt 2
resource "aws_sns_topic_subscription" "lambda"{
  topic_arn = aws_sns_topic.default.arn
  protocol  = "lambda"
  endpoint  = aws_lambda_function.create_lambda.arn
}

//Gateway
resource "aws_lambda_permission" "lambda_api" {
statement_id  = "AllowExecutionFromGateway"
action        = "lambda:InvokeFunction"
function_name = aws_lambda_function.create_lambda.function_name
principal     = "apigateway.amazonaws.com"
source_arn    = "arn:aws:lambda:${var.region}:${data.aws_caller_identity.current.account_id}:${var.gateway_id}"

}

//API integration with Lambda setup
resource "aws_api_gateway_integration" "lambda_getaway" {
  rest_api_id             = "${aws_api_gateway_rest_api.ApiCreate.id}"
  resource_id             = "${aws_api_gateway_resource.gateway_resource.id}"
  http_method             = "${aws_api_gateway_method.gateway_method.http_method}"
  type                    = "AWS"
  uri                     = aws_lambda_function.create_lambda.invoke_arn
  integration_http_method = "${var.integration_http_method}"
}

//Provides an HTTP Method Integration Response for an API Gateway Resource.
resource "aws_api_gateway_integration_response" "gateway_integration_response" {
  rest_api_id = "${aws_api_gateway_rest_api.ApiCreate.id}"
  resource_id = "${aws_api_gateway_resource.gateway_resource.id}"
  http_method = "${aws_api_gateway_method.gateway_method.http_method}"
  status_code = "${aws_api_gateway_method_response.success_response.status_code}"
  depends_on  = [aws_api_gateway_integration.lambda_getaway]

  response_templates = {
    "application/json" = ""
  }
}

//data source to get access to the effective accountID, userID and ARN in which terraform is authorized
data "aws_caller_identity" "current"{}

//API Configuration 
resource "aws_api_gateway_rest_api" "ApiCreate" {
  name ="${var.gateway_path}"
  endpoint_configuration {
    types = ["REGIONAL"]
  }
}

//Provides an API Gateway Resource.
resource "aws_api_gateway_resource" "gateway_resource" {
  parent_id   = "${aws_api_gateway_rest_api.ApiCreate.root_resource_id}"
  rest_api_id = "${aws_api_gateway_rest_api.ApiCreate.id}"
  path_part   = "${var.gateway_path}"
}

//Provides a HTTP Method for an API Gateway Resource.
resource "aws_api_gateway_method" "gateway_method" {
  rest_api_id      = "${aws_api_gateway_rest_api.ApiCreate.id}"
  resource_id      = "${aws_api_gateway_resource.gateway_resource.id}"
  http_method      = "${var.gateway_method}"
  authorization    = "${var.gateway_authorization}"
}

//Provides an HTTP Method Response for an API Gateway Resource.
resource "aws_api_gateway_method_response" "success_response" {
  rest_api_id = "${aws_api_gateway_rest_api.ApiCreate.id}"
  resource_id = "${aws_api_gateway_resource.gateway_resource.id}"
  http_method = "${aws_api_gateway_method.gateway_method.http_method}"
  status_code = "200"
  
  response_models = {
    "application/json" = "Empty"
  }
}

//AWS DLQ setup
resource "aws_sqs_queue" "dlq_queue" {
  name             = "${var.dlq_queue}"
  delay_seconds    = 30     //gives delay before sqs processes message. 
  max_message_size = 262144 //max bytes for SqS client
}
