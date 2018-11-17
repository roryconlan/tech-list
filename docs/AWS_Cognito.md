# AWS Cogntio

## Contents

- [Resources](#Resources)
- [Summary](#Summary)

## Resources

- [AWS Cogntio](https://aws.amazon.com/cognito)

## Summary

Amazon Cognito is a web service that delivers scoped temporary credentials to
mobile devices and other untrusted environments. Amazon Cognito uniquely
identifies a device and supplies the user with a consistent identity over the
lifetime of an application.

[Introduction to Cognito.](
https://docs.aws.amazon.com/cognito/latest/developerguide/what-is-amazon-cognito.html)

User pools are user directories that provide sign-up and sign-in options for
your app users. Identity pools enable you to grant your users access to other
AWS services. To say that an identity pool is bound to or integrated with a
user pool means that the user pool is the Cognito Identity Provider for the
identity pool.

- In the first step your app user signs in through a user pool and receives
  user pool tokens after a successful authentication.
- Next, your app exchanges the user pool tokens for AWS credentials through an
  identity pool.
- Finally, your app user can then use those AWS credentials to access other AWS
  services such as Amazon S3 or DynamoDB.

To save user profile information, your identity pool needs to be integrated
with a user pool.

### Exercise to illustrate the relationship of Cognito user pools to identity pools

Use the AWS CLI to look at the pools created by the
[AWS reference architecture](
  https://github.com/aws-quickstart/saas-identity-cognito)
for a multi-tenant software as a service (SaaS) environment, using Amazon
Cognito as the identity provider.

    aws cognito-idp list-user-pools \
      --max-results 60 \
      --output json

For example, this one is a sysadmin user pool:

    {
      "CreationDate": 1532531538.091,
      "LastModifiedDate": 1532531538.091,
      "LambdaConfig": {},
      "Id": "eu-west-1_xRwxzdsSL",
      "Name": "SYSADMIN4c4af9069c9f4af4bbdb03fb3ae5c2ad"
    }

Finding a corresponding identity pool by user pool ID doesn't work, there is no
reference to an identity pool in the user pool description:

    aws cognito-idp describe-user-pool \
      --user-pool-id eu-west-1_b724wlV7k \
      --output json

The identity pool is bound to or integrated with a user pool. This binding
can be seen when you view the identity pool description.

    aws cognito-identity list-identity-pools \
      --max-results 60 \
      --output json

In this case the user pool and identity pool have the same name:

    {
      "IdentityPoolId": "eu-west-1:3cfccb96-e198-41d5-a49b-4235165da7e4",
      "IdentityPoolName": "SYSADMIN4c4af9069c9f4af4bbdb03fb3ae5c2ad"
    },

Now, get the details for this identity pool:

    aws cognito-identity describe-identity-pool \
      --identity-pool-id eu-west-1:3cfccb96-e198-41d5-a49b-4235165da7e4 \
      --output json

    {
      "IdentityPoolId": "eu-west-1:3cfccb96-e198-41d5-a49b-4235165da7e4",
      "AllowUnauthenticatedIdentities": false,
      "CognitoIdentityProviders": [
        {
          "ServerSideTokenCheck": true,
          "ClientId": "48g0fnr5780q3rgvsktphgb7gc",
          "ProviderName": "cognito-idp.eu-west-1.amazonaws.com/eu-west-1_xRwxzdsSL"
        }
      ],
      "IdentityPoolName": "SYSADMIN4c4af9069c9f4af4bbdb03fb3ae5c2ad"
    }

The user pool with ID eu-west-1_xRwxzdsSL is the Cognito Identity Provider for
the Identity pool with ID eu-west-1:3cfccb96-e198-41d5-a49b-4235165da7e4. Both
the user pool and identity pool have the same name to illustrate this binding.

This can also be viewed in the console by browsing to the identity pool. Under
"Authentication providers" you can see the corresponding user pool ID.

Here you can configure your identiy pool to accept users federated
(authenticated) with your user pool and\or other public identity providers.
