# AWS Lambda

## Contents

- [Resources](#Resources)
- [Summary](#Summary)
- [Auth](#Auth)
- [FAQ](#FAQ)

## Resources

- [resource](https://link)

## Summary

[source](https://link)

## Auth

### Lambda Authentication and Authorisation

Each Lambda function has an execution role that determines the capabilities of
the function (e.g., which AWS services it can call). The account that owns the
Lambda function (and thus controls its behavior) is not necessarily the same as
the role/user that calls the function. For clarity, we’ll distinguish invocation
(and the invocation role when calling the Lambda API) from execution
([ref.](https://aws.amazon.com/blogs/compute/easy-authorization-of-aws-lambda-functions/)).

Definitions:

- Policy: A policy is a set of capabilities. It answers the "who can do what"
  question.
- Role-based Authorization: In this approach to authorization, policies are
  attached to real users and temporary or simulated users (roles). The policy
  defines what the role can do, and services enforce that the policy terms are
  met. For example, if a user named Joe has a policy that lets him create Lambda
  functions, then Lambda will check this privilege each time Joe attempts to
  make a new function and will allow that activity to proceed.
- Resource-based Authorization: Policies can also be attached to resources,
  such as Lambda functions. While not required, resource policies also often
  refer to "foreign" resources, such as restricting the set of S3 buckets that
  are allowed to send events to a specific Lambda function. Resource policies
  are especially useful in authorizing such "on-behalf-of" activities and for
  enabling cross-account access.

### Calling a Lambda function from a web service in the same account

This is an example of Role-based Authorization.
In this scenario the signer of the request determines the identity (user or
role) of the invoker, and that in turn identifies one or more policies that
specify what’s allowed. The user (or role) calling Lambda needs permission to
invoke the function. Add the following policy to the user's role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": ["lambda:InvokeFunction"],
      "Effect": "Allow",
      "Resource": "arn:aws:lambda:us-east-1:<account number>:<function name>"
    }
  ]
}
```

### Calling a Lambda function from a web service in a different account

This is an example of Resource-based Authorization.

This is the same as the scenario above except that the account of the invoker
and the account of the Lambda function owner are different. In this case
resource policies are the easiest way to authorize the activity: The resource
policy on the function being called will enable access by the “foreign” account
of the caller. Here’s a sample of how to authorize account 012345678912 to call
“MyFunction” from the command line:

```bash
$ aws lambda add-permission \
  --function-name MyFunction \
  --region us-west-2 \
  --statement-id Id-123 \
  --action "lambda:InvokeFunction" \
  --principal 012345678912
```

Note that the user (or role) making the call still needs permission to invoke
the Lambda function as in the previous example. Think of this as an “and”
condition: As a user (or role) you’ll need permission to call Lambda functions
AND you need the function owner’s permission to use his or her particular
function.

### Triggering a Lambda from an S3 bucket notification in another account

In the previous example the call from another account was made directly to
Lambda. In this scenario the call is indirect: S3 sends the event on behalf of
the bucket owner instead of that account making the call itself. The
add-permission call is slightly different:

```bash
$ aws lambda add-permission \
  --function-name MyFunction \
  --region us-west-2 \
  --statement-id Id-123 \
  --action "lambda:InvokeFunction" \
  --principal s3.amazonaws.com \
  --source-arn arn:aws:s3:::<source-bucket> \
  --source-account <account number>
```

Note that the principal becomes the S3 service (it gets a service name as
opposed to an account number) and the actual account number moves to the
“source-account” parameter. There’s also a new argument, source-arn, which
contains the name of the bucket. This mechanism gives you great flexibility in
how (and what) you choose to authorize S3 to do on your behalf.

An example of a policy:

```json
{
   "Policy":{
      "Version":"2012-10-17",
      "Statement":[
         {
            "Effect":"Allow",
            "Principal":{
               "Service":"s3.amazonaws.com"
            },
            "Action":"lambda:InvokeFunction",
            "Resource":"arn:aws:lambda:region:account-id:function:HelloWorld",
            "Sid":"65bafc90-6a1f-42a8-a7ab-8aa9bc877985",
            "Condition":{
               "StringEquals":{
                  "AWS:SourceAccount":"account-id"
               },
               "ArnLike":{
                  "AWS:SourceArn":"arn:aws:s3:::ExampleBucket"
               }
            }
         }
      ]
   }
}
```

The condition ensures that the bucket where the event occurred is owned by the
same account that owns the Lambda function.

### API Gateway Lambda Authorizers ([ref.](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-use-lambda-authorizer.html))

An Amazon API Gateway Lambda authorizer (formerly known as a custom authorizer)
is a Lambda function that you provide to control access to your API. A Lambda
authorizer uses bearer token authentication strategies, such as OAuth or SAML.
It can also use information described by headers, paths, query strings, stage
variables, or context variables request parameters.

Path parameters can be used to grant or deny permissions to invoke a method,
but they cannot be used to define identity sources, which can be used as parts
of an authorization policy caching key. Only headers, query strings, stage
variables, and context variables can be set as identity sources.

When a client calls your API, API Gateway verifies whether a Lambda authorizer
is configured for the API method. If so, API Gateway calls the Lambda function.
In this call, API Gateway supplies the authorization token that is extracted
from a specified request header for the token-based authorizer, or passes in
the incoming request parameters as the input (for example, the event parameter)
to the request parameters-based authorizer function.

## FAQ
