# AWS AppSync

## Contents

- [Resources](#Resources)
- [AppSync concepts](#AppSync-concepts)
- [Sample app](#Sample-app)
- [Test and Debug Resolvers](#Test-and-Debug-Resolvers)
- [Pipeline Resolvers](#Pipeline-Resolvers)
- [Security](#Security)
- [Authorisation Use Cases](#Authorisation-Use-Cases)
- [VTL Resolver Mapping Templates](#VTL-Resolver-Mapping-Templates)
- [Context map](#Context-map)
- [Fingerprinting AppSync files on S3](#Fingerprinting-AppSync-files-on-S3)
- [Troubleshooting and Common Mistakes](#Troubleshooting-and-Common-Mistakes)
- [FAQ](#FAQ)

## Resources

- [AWS AppSync](https://aws.amazon.com/appsync)
  
  Reference documentation:
  - [Context map](#Context-map)
  - [Util helpers](
    https://docs.aws.amazon.com/appsync/latest/devguide/resolver-util-reference.html)

    AppSync helpers for GraphQL resolvers to simplify interactions with data sources.
  - [DynamoDB](
    https://docs.aws.amazon.com/appsync/latest/devguide/resolver-mapping-template-reference-dynamodb.html)

    Use GraphQL to store and retrieve data in existing DynamoDB tables.
  - [ElasticSearch](
    https://docs.aws.amazon.com/appsync/latest/devguide/resolver-mapping-template-reference-elasticsearch.html)

    Use GraphQL to store and retrieve data in existing ElasticSearch domains.
  - [Lambda](
    https://docs.aws.amazon.com/appsync/latest/devguide/resolver-mapping-template-reference-lambda.html)

    Resolve GraphQL requests with Lambda functions.
  - [NONE data source](
    https://docs.aws.amazon.com/appsync/latest/devguide/resolver-mapping-template-reference-none.html)

    Passing the output of the request template directly to the response template.
  - [HTTP](
    https://docs.aws.amazon.com/appsync/latest/devguide/resolver-mapping-template-reference-http.html)

    The HTTP resolver mapping templates enable you to shape requests from
    AppSync to any HTTP endpoint, and responses from the endpoint back to
    AppSync.
  - [Monitoring and Logging](
    https://docs.aws.amazon.com/appsync/latest/devguide/monitoring.html)
- [GraphQL](./GraphQL.md)
- [Apache Velocity Template Language (VTL)](
  http://velocity.apache.org/engine/2.0/vtl-reference.html)
- Tutorials:
  - [Client app](
    https://docs.aws.amazon.com/appsync/latest/devguide/building-a-client-app.html)
  - [Data sources and Resolvers](
    https://docs.aws.amazon.com/appsync/latest/devguide/tutorials.html)
  - [Serverless GraphQL with AWS AppSync and Go Lambda functions](
    https://sbstjn.com/serverless-graphql-with-appsync-and-lambda.html)
- Articles:
  - [How developers can authenticate and authorise users with AppSync](
    https://read.acloud.guru/authentication-and-authorization-with-aws-appsync-ccbd77658de2)
  - [Curated list of AWS AppSync Resources](
    https://github.com/dabit3/awesome-aws-appsync)

## AppSync concepts

AWS AppSync enables developers to interact with their data by using a managed
GraphQL service.

### GraphQL Proxy

A component that runs the GraphQL engine for processing requests and mapping
them to logical functions for data operations or triggers. The data resolution
process performs a batching process (called the Data Loader) to your data
sources. This component also manages conflict detection and resolution
strategies.

### Operation

AWS AppSync supports the three GraphQL operations: query (read-only fetch),
mutation (write followed by a fetch), and subscription (long-lived requests
that receive data in response to events).

### Action

There is one action that AWS AppSync defines. This action is a notification to
connected subscribers, which is the result of a mutation. Clients become
subscribers through a handshake process following a GraphQL subscription.

### Data Source

A persistent storage system or a trigger, along with credentials for accessing
that system or trigger. Your application state is managed by the system or
trigger defined in a data source.

### Resolver

A function that converts the GraphQL payload to the underlying storage system
protocol and executes if the caller is authorised to invoke it. Resolvers are
comprised of request and response mapping templates, which contain
transformation and execution logic. They use mapping templates written in
[Apache Velocity Template Language (VTL)](
http://velocity.apache.org/engine/2.0/vtl-reference.html)
to convert a GraphQL expression into a format the data source can use.

These templates enable you to customize the behavior, and apply logic and
conditions before and after communicating with the data source.

#### Resolver components

- The location in the GraphQL schema to attach the resolver.
- The data source to use for this resolver.
- The request mapping template.
- The response mapping template.

Example flow:

- AppSync receives an addPost mutation request.
- AppSync takes the request, and the request mapping template, and generates
  a request mapping document.
- AppSync uses the request mapping document to generate and execute a
  DynamoDBPutItem request.
- AppSync takes the results of the PutItem request and converts them back to
  GraphQL types.
- Passes it through the response mapping document, which just passes it
  through unchanged.
- Returns the newly created object in the GraphQL response.

Notes:

- $ctx is just an alias for $context. When testing you can use a mock context
  object to see the transformed value of the template before invoking.
  See [here](
  https://docs.aws.amazon.com/appsync/latest/devguide/resolver-context-reference.html#aws-appsync-resolver-mapping-template-context-reference)
  for the structure of context and the available helper utilities.
  The tests and mock context objects seem to be all done in the console.
  Wonder if there's a programmatic way to run them?
- A type is specified on all the keys and attribute values. For example, you
  set the author field to { "S" : "${context.arguments.author}" }. The S part indicates to AWS AppSync and DynamoDB that the value will be a string value.

      "author": { "S" : "${context.arguments.author}" },

  Similarly, an N type is a number field.
- AWS AppSync comes with a utility for automatic ID generation called
  $utils.autoId() which you could have also used in the form of "id" : { "S" : "${$utils.autoId()}" }.
- A Post type need not be a flat key/value object. You can also model complex
  objects with the AWS AppSyncDynamoDB resolver, such as sets, lists, and maps.
- In a response mapping template, if the required shape of the data in your
  table exactly matches the shape of the Post type in GraphQL, the response mapping template can pass the results straight through:

      $utils.toJson($context.result)
- An update of an item is significantly different from a PutItem operation.
  Instead of writing the entire item, we're just asking DynamoDB to update
  certain attributes. This is done using DynamoDB Update Expressions. The
  expression itself is specified in the expression field in the update
  section. The values to use do not appear in the expression itself; the
  expression has placeholders that have names starting with a colon, which are
  then defined in the expressionValues field. Finally, DynamoDB has reserved
  words that cannot appear in the expression. For example, url is a reserved
  word, so to update the url field we can use name placeholders and define
  them in the expressionNames field.

      {
          "version" : "2017-02-28",
          "operation" : "UpdateItem",
          "key" : {
              "id" : { "S" : "${context.arguments.id}" }
          },
          "update" : {
              "expression" : "SET author = :author, title = :title, content = :content, #url = :url ADD version :one",
              "expressionNames": {
                  "#url" : "url"
              },
              "expressionValues": {
                  ":author" : { "S": "${context.arguments.author}" },
                  ":title" : { "S": "${context.arguments.title}" },
                  ":content" : { "S": "${context.arguments.content}" },
                  ":url" : { "S": "${context.arguments.url}" },
                  ":one" : { "N": 1 }
              }
          }
      }

  Optimistic locking pattern:

  The above UpdateItem has two main problems:
  - If you want to update just a single field, you have to update all of the
  fields.
  - If two people are modifying the object, you could potentially lose information.

  AppSync solves this using the optimistic locking pattern
  [(ref.)](
    https://docs.aws.amazon.com/appsync/latest/devguide/tutorial-dynamodb-resolvers.html).

  This pattern adds code to the resolver to iterate through each argument,
  skipping "id" and "expectedVersion". If the argument is set to "null", then
  the attribute is removed from the item in DynamoDB. Otherwise set (or update)
  the attribute on the item in DynamoDB. If an argument isn't specified, it
  leaves the attribute alone. It also increments the version field.

  Also, there is a new condition section. A condition expression enables you
  tell AppSync and DynamoDB whether or not the request should succeed based on
  the state of the object already in DynamoDB before the operation is
  performed. In this case, we only want the UpdateItem request to succeed if
  the version field of the item currently in DynamoDB exactly matches the
  expectedVersion argument.

  A feature of an AppSync DynamoDB resolver is that it returns the current
  value of the post object in DynamoDB. You can find this in the data field in
  the errors section of the GraphQL response if the condition expression fails.
  Your application can use this information to decide how it should proceed.
  For example, you could increment the expectedVersion argument and try again.

- AWS Lambda Resolvers:

  AppSync enables you to use Lambda functions to resolve any GraphQL field.
  For example, a GraphQL query might send a call to an Amazon RDS instance,
  and a GraphQL mutation might write to a Amazon Kinesis stream. A Lambda
  function performs the business logic. This can also mean that the resolver
  mapping templates contain less logic. Errors can be raised in the Lambda
  function as well as in the resolver mapping templates.

  E.g.: The use of $utils.appendError() is similar to the $util.error(), with
  the major distinction that it doesn't interrupt the evaluation of the
  mapping template. Instead, it signals there was an error with the field, but
  allows the template to be evaluated and consequently return data back to the
  caller. It is recommended you use $utils.appendError() when your application
  needs to return partial results. On the other hand, $util.error(...) aborts
  the template execution without returning data.

- Local Resolvers:

  These are used when a call to a data source might not be necessary. Instead
  of calling a remote data source, the local resolver will just forward the
  result of the request mapping template to the response mapping template. The
  field resolution will not leave AWS AppSync.

  Local resolvers are useful for several use cases. The most popular use case is to publish notifications without triggering a data source call.

- DynamoDB Batch Resolvers

  AppSync supports using DynamoDB batch operations across one or more tables
  in a single region. Supported operations are BatchGetItem, BatchPutItem, and
  BatchDeleteItem:

  - Pass a list of keys in a single query and return the results from a table.
  - Read records from one or more tables in a single query.
  - Write records in bulk to one or more tables.
  - Conditionally write or delete records in multiple tables that might have a
    relation
  
  Batch operations require that:
  - The data source role must have permissions to all tables which the
    resolver will access.
  - The table specification for a resolver is part of the mapping template.

- HTTP Resolvers

  AppSync enables you to use any arbitrary HTTP endpoints to resolve GraphQL
  fields. Once your HTTP endpoints are set up, you can connect to them as a
  data source. Then, you can configure a resolver in the schema to perform
  GraphQL operations such as queries, mutations, and subscriptions. E.g. a
  REST API (created using Amazon API Gateway and Lambda) with an AppSync
  GraphQL endpoint in front of it. You're basically translating REST format
  queries to GraphQL.

  Note:
  - At this time only endpoints that are publicly accessible are supported by
    AppSync.
  - If your data source is an HTTPS endpoint, the endpoint must have a server
    certificate signed by a trusted certificate authority (CA) (self-signed
    certificates are not supported at this time). AppSync is only be able to
    talk to HTTPS endpoints that have a signed certificate from a trusted CA
    that it recognises.

### Subscriptions

Subscriptions in AppSync are invoked as a response to a GraphQL mutation.
You configure this with a Subscription type and @aws_subscribe() directive in
the schema to denote which mutations invoke one or more subscriptions.

[Real time data](
https://docs.aws.amazon.com/appsync/latest/devguide/real-time-data.html)

This means that you can make any data source in AppSync real time by
specifying a GraphQL schema directive on a mutation. Subscription connection
management is handled automatically by the AppSync client SDK using MQTT over
WebSockets as the network protocol between the client and service.

Note: To control authorisation at connection time to a subscription, you can
leverage controls such as IAM, Amazon Cognito identity pools, or Amazon
Cognito user pools for field-level authorisation. For fine-grained access
controls on subscriptions, you can attach resolvers to your subscription
fields and perform logic using the identity of the caller and AppSync data
sources.

Subscriptions are triggered from mutations and the mutation selection set is
sent to subscribers. As the client app triggers the mutation, it defines the
set that it will receive every time the subscription is updated.

### Identity (of the caller)

A representation of the caller based on a set of credentials, which must be
sent along with every request to the GraphQL proxy. It includes permissions to
invoke resolvers. Identity information is also passed as context to a resolver
and the conflict handler to perform additional checks.

### AWS AppSync Client

The location where GraphQL operations are defined. The client performs
appropriate authorisation wrapping of request statements before submitting to
the GraphQL proxy. Responses are persisted in an offline store and mutations
are made in a write-through pattern.

With AppSync you can start with a schema and auto-provision a data source and
resolvers. You can also start with a DynamoDB to auto-provision a schema and
resolvers! There is also a guided wizard for quickly creating schemas.

### Error Handling

In AppSync, data source operations can sometimes return partial results.
Partial results is the term we will use to denote when the output of an
operation is comprised of some data and an error. Because error handling is
inherently application specific, AppSync gives you the opportunity to handle
errors in the response mapping template. The resolver invocation error, if
present, is available from the context as $ctx.error. Invocation errors always
include a message and a type, accessible as properties $ctx.error.message and
$ctx.error.type. During the response mapping template invocation, you can
handle partial results in three ways:

- swallow the invocation error by just returning data
- raise an error (using $util.error(...)) by stopping the response mapping
  template evaluation, which won't return any data.
- append an error (using $util.appendError(...)) and also return data

See [here](
  https://docs.aws.amazon.com/appsync/latest/devguide/tutorial-dynamodb-batch.html#error-handling).

## Sample app

[AWS sample app.](
https://github.com/aws-samples/aws-mobile-appsync-events-starter-react)

This is a Starter React application for using the Sample app in the AWS
AppSync console when building your GraphQL API. The Sample app creates a
GraphQL schema and provisions Amazon DynamoDB resources, then connects them
appropriately with Resolvers. The application demonstrates GraphQL Mutations,
Queries and Subscriptions using AWS AppSync. You can use this for learning
purposes or adapt either the application or the GraphQL Schema to meet your
needs.

Nice pattern:

- Components folder:
  Contains component files for each operation e.g. NewEvent.js.
- GraphQL folder:
  Contains the gql part for each operation e.g. MutationCreateEvent.js wraps
  the GraphQL call in js using the graphql-tag library.

## Test and Debug Resolvers

See [here](
https://docs.aws.amazon.com/appsync/latest/devguide/test-debug-resolvers.html).

AppSync executes resolvers on a GraphQL field against a data source. To help
developers write, test, and debug these resolvers, the AppSync console also
provides tools to create a GraphQL request and response with mock data, down to
the individual field resolver. Additionally, you can perform queries, mutations,
and subscriptions in the AppSync console, and see a detailed log stream from
CloudWatch of the entire request. This includes results from a data source.

### Testing with Mock Data

When a GraphQL resolver is invoked, it contains a context object that has
relevant information about the request for you to program against. This includes
arguments from a client, identity information, and data from the parent GraphQL
field. It also has results from the data source, which can be used in the
response template.

When writing or editing a resolver function, you can pass a mock or test context
object into the console editor. This enables you to see how both the request and
the response templates evaluate without actually running against a data source.

### Test a Resolver

In the AppSync console, go to the Schema page, and choose an existing resolver
on the right to edit it. Or, choose Attach to add a new resolver. At the top of
the page, choose Select test context, choose Create new context, and then enter
a name. Next, either select from an existing sample context object or populate
the JSON manually, and then choose Save. To evaluate your resolver using this
mocked context object, choose Run Test.

For example, suppose you have an app storing a GraphQL type of Dog that uses
automatic ID generation for objects and stores them in DynamoDB. You also want
to write some values from the arguments of a GraphQL mutation, and allow only
specific users to see a response. The following shows what the schema might look
like:

    type Dog {
      breed: String
      color: String
    }

    type Mutation {
      addDog(firstname: String, age: Int): Dog
    }

When you add a resolver for the addDog mutation, you can populate a context
object like the one following. The following has arguments from the client of
name and age, and a username populated in the identity object:

    {
      "arguments" : {
        "firstname": "Shaggy",
        "age": 4
      },
      "source" : {},
      "result" : {
        "breed" : "Miniature Schnauzer",
        "color" : "black_grey"
      },
      "identity": {
        "sub" : "uuid",
        "issuer" : "https://cognito-idp.region.amazonaws.com/userPoolId",
        "username" : "Nadia",
        "claims" : { },
        "sourceIP" : "x.x.x.x",
        "defaultAuthStrategy" : "ALLOW"
      }
    }

You can test this using the following request and response mapping templates:

Request Template

    ## When running a resolver test in the console you can put stuff in here
    ## to print out the contents of variables at runtime. E.g.:
    #set($userInfo = {})
    #set($userInfo.userName = $context.identity.username)
    #set($userInfo.userId = $context.identity.sub)

    ## Then print it out:
    $userInfo

    ## Print out other stuff to see what's going on:
    $util.autoId()
    $util.dynamodb.toMapValuesJson($ctx.args)

    {
      "version" : "2017-02-28",
      "operation" : "PutItem",
      "key" : {
        "id" : { "S" : "$util.autoId()" }
      },
      "attributeValues" : $util.dynamodb.toMapValuesJson($ctx.args)
    }

Response Template

    #if ($context.identity.username == "Nadia")
      $util.toJson($ctx.result)
    #else
      $util.unauthorized()
    #end

The evaluated template has the data from your test context object and the
generated value from $util.autoId(). Additionally, if you were to change the
username to a value other than Nadia, the results won't be returned because the
authorization check would fail.

### Debugging a Live Query

There's no substitute for an end-to-end test and logging to debug a production
application. AppSync lets you log errors and full request details using
CloudWatch. Additionally, you can use the AppSync console to test GraphQL
queries, mutations, and subscriptions and live stream log data for each request
back into the query editor to debug in real time. For subscriptions, the logs
display connection-time information.

To perform this, you need to have CloudWatch logs enabled in advance. Next, in
the AppSync console, choose the Queries tab and then enter a valid GraphQL
query. In the lower-right section, select the Logs check box to open the logs
view. At the top of the page, choose the play arrow icon to run your GraphQL
query. In a few moments, your full request and response logs for the operation
are streamed to this section and you can view then in the console.

## Pipeline Resolvers

AppSync executes resolvers on a GraphQL field. In some cases, applications
require executing multiple operations to resolve a single GraphQL field. With
pipeline resolvers, developers can now compose operations (called Functions) and
execute them in sequence. Pipeline resolvers are useful for applications that,
for instance, require performing an authorisation check before fetching data for
a field.

### Anatomy of a pipeline resolver

A pipeline resolver is composed of a Before mapping template, an After mapping
template, and a list of Functions. Each Function has a request and response
mapping template that it executes against a Data Source. As a pipeline resolver
delegates execution to a list of functions, it is therefore not linked to any
data source. Unit resolvers and functions are primitives that execute operation
against data sources.

So, your request mapping template feeds data to a function, which can feed
another function, and so on until it finally feeds the response mapping
template.

See [here](
https://docs.aws.amazon.com/appsync/latest/devguide/pipeline-resolvers.html)
for more information.

## Security

See [here](https://docs.aws.amazon.com/appsync/latest/devguide/security.html).

- API_KEY Authorisation

  An API key is a hard-coded value in your application that is generated by
  the AppSync service when you create an unauthenticated GraphQL endpoint. You
  can rotate API keys from the console, from the CLI, or from the AppSync API
  Reference. They are recommended for development purposes or use cases where it's safe to expose a public API.

- AWS_IAM Authorisation

  This authorisation type enforces the [AWS Signature Version 4 Signing
  Process](
  https://docs.aws.amazon.com/general/latest/gr/signature-version-4.html)
  on the GraphQL API. You can associate Identity and Access Management (IAM)
  access policies with this authorisation type. Your application can leverage
  this association by using an access key (which consists of an access key ID
  and secret access key) or by using short-lived, temporary credentials
  provided by Amazon Cognito Federated Identities.

- OPENID_CONNECT Authorisation

  This authorisation type enforces OpenID Connect (OIDC) tokens provided by an
  OIDC-compliant service. Your application can leverage users and privileges
  defined by your OIDC provider for controlling access.

  An Issuer URL is the only required configuration value that you provide to
  AppSync e.g. auth.example.com. This URL must be addressable over HTTPS.
  AppSync appends /.well-known/openid-configuration to the issuer URL
  to locate the OpenID configuration (per the OpenID Connect Discovery
  specification). It expects to retrieve an RFC5785 compliant JSON document at
  this URL. This JSON document must contain a jwks_uri key, which points to
  the JSON Web Key Set (JWKS) document with the signing keys.

  AppSync requires the JWKS to contain JSON fields of alg, kty, and kid.

- AMAZON_COGNITO_USER_POOLS Authorisation

  This authorisation type enforces OIDC tokens provided by Amazon Cognito User
  Pools. Your application can leverage the users and groups in your user pools
  and associate these with GraphQL fields for controlling access.

  When using Cognito User Pools, you can create groups that users belong to.
  This information is encoded in a JWT token that your application sends to
  AppSync in an authorisation header when sending GraphQL operations. You can
  use GraphQL directives on the schema to control which groups can invoke
  which resolvers on a field, thereby giving more controlled access to your
  customers.
  
  As an example, you only want bloggers to be able to create new posts:

      schema {
        query: Query
        mutation: Mutation
      }

      type Query {
        posts:[Post!]!
        @aws_auth(cognito_groups: ["Bloggers", "Readers"])
      }

      type Mutation {
        addPost(id:ID!, title:String!):Post!
        @aws_auth(cognito_groups: ["Bloggers"])
      }

  Note that you can omit the @aws_auth directive if you want to default to a
  specific grant-or-deny strategy on access. You can specify the grant-or-deny
  strategy in the user pool configuration when you create your GraphQL API.

- Fine-Grained Access Control

  The preceding information demonstrates how to restrict or grant access to
  certain GraphQL fields. If you want to set access controls on the data itself
  based on certain conditions - such as who the user is that is making a call
  and whether they own the data - you can do so using mapping templates in your
  resolvers.

  Say you want to go further in the above example such that only users that
  created a post are allowed to edit it. The user would sign into an app and
  get credentials from an IDP, such as Cognito User Pools, and then pass these
  credentials to AppSync as part of a GraphQL operation. The mapping template
  would then substitute a value from the credentials (like the user ID) in a
  conditional statement which will then be compared to a value in your database.

  The new GraphQL operation would be:

      type Mutation {
        editPost(id:ID!, title:String, content:String):Post
      }

  Since this is an edit operation, it corresponds to an UpdateItem in DynamoDB.
  You can perform a conditional check before performing this action. The
  identity information is passed from the GraphQL operation in the context
  object (context.identity).

  Here is an example of the request mapping template for editPost, which only
  updates the content of the blog post if the request comes from the user that
  created the post:

      {
        "version" : "2017-02-28",
        "operation" : "PutItem",
        "key" : {
          "id": $util.dynamodb.toDynamoDBJson($ctx.args.id),
        },
        "attributeValues" : $util.dynamodb.toMapValuesJson($ctx.args),
        "condition" : {
          "expression" : "contains(#author,:expectedOwner)",
          "expressionNames" : {
            "#author" : "Author"
          },
          "expressionValues" : {
            ":expectedOwner" : { "S" : "${context.identity.username}" }
          }
        }
      }

  This example uses a PutItem that overwrites all values rather than an
  UpdateItem, which would be a bit more verbose in an example, but the same
  concept applies on the condition statement block.

- Filtering Information

  Whereas the above is a request mapping template, you can also perform logic
  in the response mapping template. For example, the following will filter the
  response:

      {
        #if($context.result["Author"] == "$context.identity.username")
          $utils.toJson($context.result);
        #end
      }

  If the condition is false, a null response is returned.

- Data source access

  AppSync communicates with data sources using Identity and Access Management
  (IAM) roles and access policies. If you are using an existing role, a Trust
  Policy needs to be added in order for AWS AppSync to assume the role. The
  trust relationship will look like below:

      {
        "Version": "2012-10-17",
        "Statement": [
          {
            "Effect": "Allow",
            "Principal": {
              "Service": "appsync.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
          }
        ]
      }

  It's important to scope down the access policy on the role to only have
  permissions to act on the minimal set of resources necessary.

## Authorisation Use Cases

This section goes into detail on how to protect your data using GraphQL
resolver mapping templates. You can protect data on read or write in a very
flexible manner using a combination of user identity, conditionals, and data
injection.

AppSync passes identity (user/role) information into the GraphQL request and
response as a context object, which you can use in the resolver. This means
that permissions can be granted appropriately either on write or read
operations based on the resolver logic. If this logic is at the resource
level, for example only certain named users or groups can read/write to a
specific database row, then that "authorisation metadata" must be stored.
I.e. a user ID must be stored in a DynamoDB table so that it can be compared
to the user ID passed in from Cognito in the context object. AppSync does not
store any data so therefore you must store this authorisation metadata with
the resources so that permissions can be calculated. Authorisation metadata
is usually an attribute (column) in a DynamoDB table, such as an owner or
list of users/groups. For example there could be Readers and Writers
attributes.

Note: For both request and response templates you can also use custom headers from clients to perform validation checks.

### Reading Data

As outlined above the authorisation metadata to perform a check must be stored
with a resource or passed in to the GraphQL request (identity, header, etc.).

To demonstrate this suppose you have the DynamoDB table below:
![table](https://docs.aws.amazon.com/appsync/latest/devguide/images/auth.png)

The primary key is ID and the data to be accessed is Data. The other columns
are examples of checks you can perform for authorisation. Owner would be a
String while PeopleCanAccess and GroupsCanAccess would be String Sets as
outlined in the Resolver Mapping Template Reference for DynamoDB.

Read single item:

- For GraphQL queries of individual items, you can use the response template to
  check if the user is allowed to see these results or return an authorisation
  error message. This is sometimes referred to as an "authorisation filter".
  E.g. you perform a conditional #if () ... #end statement in the response
  template after the resolver has read from the data source. The check will
  normally be using user or group values in $context.identity for membership
  checks against the authorisation metadata returned from the read operation.

Read multiple items:

- For GraphQL queries returning lists, using a Scan or Query, it is more
  performant to perform the conditional check on the request template and
  return data only if an authorisation condition is satisfied. E.g. the
  authorisation check is a "filter":{"expression":...} statement. Common checks
  are equality (attribute = :input) or checking if a value is in a list
  (contains(attribute, :input)).

#### Use Case: Owner Can Read

Using the table above, if you only wanted to return data if Owner == Nadia for
an individual read operation (GetItem) your response template would look like:

    #if($context.result["Owner"] == $context.identity.username)
      $utils.toJson($context.result);
    #else
      $utils.unauthorised()
    #end

A couple things to mention here which will be re-used in the remaining
sections. First, the check uses $context.identity.username which will be the
friendly user sign-up name if Amazon Cognito user pools is used and will be the
user identity if AWS IAM is used (including Amazon Cognito Federated
Identities). There are other values to store for an owner such as the unique
"Amazon Cognito identity" value, which is useful when federating logins from
multiple locations, and you should review the options available in the Resolver
Mapping Template Context Reference.

Second, the conditional else check responding with $util.unauthorised() is
completely optional but recommended as a best practice when designing your
GraphQL API.

#### Use Case: Hardcode Specific Access

    // This checks if the user is part of the Admin group and makes the call
    #foreach($group in $context.identity.claims.get("cognito:groups"))
        #if($group == "Admin")
            #set($inCognitoGroup = true)
        #end
    #end
    #if($inCognitoGroup)
    {
        "version" : "2017-02-28",
        "operation" : "UpdateItem",
        "key" : {
            "id" : { "S" : "${context.argument.id}" }
        },
        "attributeValues" : {
            "owner" : {"S" : "${context.identity.username}"}
            #foreach( $entry in $context.arguments.entrySet() )
                ,"${entry.key}" : { "S" : "${entry.value}" }
            #end
        }
    }
    #else
        $utils.unauthorised()
    #end

#### Use Case: Filtering a List of Results

In the previous example you were able to perform a check against
$context.result directly as it returned a single item, however some operations
like a scan will return multiple items in $context.result.items where you need
to perform the authorisation filter and only return results that the user is
allowed to see. Suppose the Owner field had the Amazon Cognito IdentityID this
time set on the record, you could then use the following response mapping
template to filter to only show those records that the user owned:

    #set($myResults = [])
    #foreach($item in $context.result.items)
        ##For userpools use $context.identity.username instead
        #if($item.Owner == $context.identity.cognitoIdentityId)
            #set($added = $myResults.add($item))
        #end
    #end
    $utils.toJson($myResults)

#### Use Case: Multiple People Can Read

Another popular authorisation option is to allow a group of people to be able
to read data. In the example below the "filter":{"expression":...} only returns
values from a table scan if the user running the GraphQL query is listed in the
set for PeopleCanAccess.

    {
      "version" : "2017-02-28",
      "operation" : "Scan",
      "limit": #if(${context.arguments.count}) "${context.arguments.count}" #else 20 #end,
      "nextToken": #if(${context.arguments.nextToken}) "${context.arguments.nextToken}" #else null #end,
      "filter":{
        "expression": "contains(#peopleCanAccess, :value)",
        "expressionNames": {
          "#peopleCanAccess": "peopleCanAccess"
        },
        "expressionValues": {
          ":value": { "S" : "${context.identity.username}" }
        }
      }
    }

#### Use Case: Group Can Read

Similar to the last use case, it may be that only people in one or more groups
have rights to read certain items in a database. Use of the "expression":
"contains()" operation is similar however it's a logical-OR of all the groups
that a user might be a part of which needs to be accounted for in the set
membership. In this case we build up a $expression statement below for each
group the user is in and then pass this to the filter:

    #set($expression = "")
    #set($expressionValues = {})
    #foreach($group in $context.identity.claims.get("cognito:groups"))
      #set( $expression = "${expression} contains(groupsCanAccess, :var$foreach.count )" )
      #set( $val = {})
      #set( $test = $val.put("S", $group))
      #set( $values = $expressionValues.put(":var$foreach.count", $val))
      #if ( $foreach.hasNext )
      #set( $expression = "${expression} OR" )
      #end
    #end
    {
      "version" : "2017-02-28",
      "operation" : "Scan",
      "limit": #if(${context.arguments.count}) "${context.arguments.count}" #else 20 #end,
      "nextToken": #if(${context.arguments.nextToken}) "${context.arguments.nextToken}" #else null #end,
      "filter":{
        "expression": "$expression",
        "expressionValues": $utils.toJson($expressionValues)
      }
    }

### Writing Data

Writing data on mutations (like a PutItem or UpdateItem) is always controlled
on the request mapping template, i.e. before executing the write operation. In
the case of DynamoDB data sources, the key is to use an appropriate "condition":
{"expression"...}" which performs validation against the authorisation metadata
in that table. See the PutItem example above. The condition will often use a
value in $context.identity to compare against authorisation metadata stored on
the resource.

#### Use Case: Multiple Owners

Using the example table diagram from earlier, suppose the PeopleCanAccess list:

    {
      "version" : "2017-02-28",
      "operation" : "UpdateItem",
      "key" : {
        "id" : { "S" : "${context.arguments.id}" }
      },
      "update" : {
        "expression" : "SET meta = :meta",
        "expressionValues": {
          ":meta" : { "S": "${context.arguments.meta}" }
        }
      },
      "condition" : {
        "expression"       : "contains(Owner,:expectedOwner)",
        "expressionValues" : {
          ":expectedOwner" : { "S" : "${context.identity.username}" }
        }
      }
    }

This can be read in English as:

- Perform an UpdateItem operation on key context.arguments.id using the update
  expression "SET meta = :meta" where :meta is context.arguments.meta. Execute
  only if the expression contains(Owner,:expectedOwner) is true where
  :expectedOwner is context.identity.username.

#### Use Case: Group Can Create New Record

    #set($expression = "")
    #set($expressionValues = {})
    #foreach($group in $context.identity.claims.get("cognito:groups"))
      #set( $expression = "${expression} contains(groupsCanAccess, :var$foreach.count )" )
      #set( $val = {})
      #set( $test = $val.put("S", $group))
      ## Here we're setting up the placeholders for the expression values
      ## similar to the previous example (see ":expectedOwner" above).
      #set( $values = $expressionValues.put(":var$foreach.count", $val))
      #if ( $foreach.hasNext )
      #set( $expression = "${expression} OR" )
      #end
    #end
    {
      "version" : "2017-02-28",
      "operation" : "PutItem",
      "key" : {
        ## If your table's hash key is not named 'id', update it here. **
        "id" : { "S" : "$context.arguments.id" }
        ## If your table has a sort key, add it as an item here. **
      },
      "attributeValues" : {
        ## Add an item for each field you would like to store to Amazon DynamoDB. **
        "title" : { "S" : "${context.arguments.title}" },
        "content": { "S" : "${context.arguments.content}" },
        "owner": {"S": "${context.identity.username}" }
      },
      "condition" : {
        "expression": "attribute_not_exists(id) AND $expression",
        "expressionValues": $utils.toJson($expressionValues)
      }
    }

#### Use Case: Group Can Update Existing Record

    #set($expression = "")
    #set($expressionValues = {})
    #foreach($group in $context.identity.claims.get("cognito:groups"))
      #set( $expression = "${expression} contains(groupsCanAccess, :var$foreach.count )" )
      #set( $val = {})
      #set( $test = $val.put("S", $group))
      #set( $values = $expressionValues.put(":var$foreach.count", $val))
      #if ( $foreach.hasNext )
      #set( $expression = "${expression} OR" )
      #end
    #end
    {
      "version" : "2017-02-28",
      "operation" : "UpdateItem",
      "key" : {
        "id" : { "S" : "${context.arguments.id}" }
      },
      "update":{
        "expression" : "SET title = :title, content = :content",
        "expressionValues": {
          ":title" : { "S": "${context.arguments.title}" },
          ":content" : { "S": "${context.arguments.content}" }
        }
      },
      "condition" : {
        "expression": "$expression",
        "expressionValues": $utils.toJson($expressionValues)
      }
    }

### Public and Private Records

With the conditional filters you can also choose to mark data as private,
public or some other Boolean check. This can then be combined as part of an
authorisation filter inside the response template. Using this check is a nice
way to temporarily hide data or remove it from view without trying to control
group membership.

For example suppose you added an attribute on each item in your DynamoDB table
called public with either a value of yes or no. The following response template
could be used on a GetItem call to only display data if the user is in a group
that has access AND if that data is marked as public:

    #set($permissions = $context.result.GroupsCanAccess)
    #set($claimPermissions = $context.identity.claims.get("cognito:groups"))

    ## Loop through the groups, if this identity belongs to any group with
    ## permissions then it is ok to return the results.
    #foreach($per in $permissions)
      #foreach($cgroups in $claimPermissions)
        #if($cgroups == $per)
          #set($hasPermission = true)
        #end
      #end
    #end

    ## Only return the results if they're marked public.
    #if($hasPermission && $context.result.public == 'yes')
      $utils.toJson($context.result)
    #else
      $utils.unauthorised()
    #end

The above code could also use a logical OR (||) to allow people to read if they
have permission to a record or if it's public:

    #if($hasPermission || $context.result.public == 'yes')
      $utils.toJson($context.result)
    #else
      $utils.unauthorised()
    #end

In general, you will find the standard operators ==, !=, &&, and || helpful
when performing authorisation checks.

### Real-Time Data

You can apply Fine Grained Access Controls to GraphQL subscriptions at the time
a client makes a subscription, using the same techniques described earlier in
this documentation. You attach a resolver to the subscription field, at which
point you can query data from a data source and perform conditional logic in
either the request or response mapping template. You can also return additional
data to the client, such as the initial results from a subscription, as long as
the data structure matches that of the returned type in your GraphQL
subscription.

#### Use Case: User Can Subscribe to Specific Conversations Only

A common use case for real-time data with GraphQL subscriptions is building a
messaging or private chat application. When creating a chat application that
has multiple users, conversations can occur between two people or among
multiple people. These might be grouped into "rooms", which are private or
public. As such, you would only want to authorise a user to subscribe to a
conversation (which could be one to one or among a group) for which they have
been granted access. For demonstration purposes, the sample below shows a
simple use case of one user sending a private message to another. The setup has
two Amazon DynamoDB tables:

- Messages table: (primary key) toUser, (sort key) id
- Permissions table: (primary key) username

GraphQL schema:

    input CreateUserPermissionsInput {
      user: String!
      isAuthorisedForSubscriptions: Boolean
    }

    type Message {
      id: ID
      toUser: String
      fromUser: String
      content: String
    }

    type MessageConnection {
      items: [Message]
      nextToken: String
    }

    type Mutation {
      sendMessage(toUser: String!, content: String!): Message
      createUserPermissions(input: CreateUserPermissionsInput!): UserPermissions
      updateUserPermissions(input: UpdateUserPermissionInput!): UserPermissions
    }

    type Query {
      getMyMessages(first: Int, after: String): MessageConnection
      getUserPermissions(user: String!): UserPermissions
    }

    type Subscription {
      newMessage(toUser: String!): Message
      @aws_subscribe(mutations: ["sendMessage"])
    }

    input UpdateUserPermissionInput {
      user: String!
      isAuthorisedForSubscriptions: Boolean
    }

    type UserPermissions {
      user: String
      isAuthorisedForSubscriptions: Boolean
    }

    schema {
      query: Query
      mutation: Mutation
      subscription: Subscription
    }

Some of the standard operations, such as createUserPermissions(), are not
covered below to illustrate the subscription resolvers, but are standard
implementations of DynamoDB resolvers. Instead, we'll focus on subscription
authorisation flows with resolvers. To send a message from one user to another,
attach a resolver to the sendMessage() field and select the Messages table data
source with the following request template:

    {
      "version" : "2017-02-28",
      "operation" : "PutItem",
      "key" : {
        "toUser" : { "S" : "${ctx.args.toUser}" },
        "id" : { "S" : "${util.autoId()}" }
      },
      "attributeValues" : {
        "fromUser" : { "S" : "${context.identity.username}" },
        "content" : { "S" : "${ctx.args.content}" },
      }
    }

In this example, we use $context.identity.username. This returns user
information for AWS Identity and Access Management or Amazon Cognito users. The
response template is a simple passthrough of $util.toJson($ctx.result).

Attach a resolver for the newMessage() subscription, using the Permissions
table as a data source and the following request mapping template:

    {
      "version": "2017-02-28",
      "operation": "GetItem",
      "key": {
        "username": $util.dynamodb.toDynamoDBJson($ctx.identity.username),
      },
    }

Then use the following response mapping template to perform your authorisation
checks using data from the Permissions table:

    #if(${context.identity.username} != ${context.arguments.toUser})
      $utils.unauthorised()
    #elseif(! ${context.result.isAuthorisedForSubscriptions})
      $utils.unauthorised()
    #else
    ##User is authorised, but we return null to continue
      null
    #end

In this case, you're doing two authorisation checks. The first ensures that the
user isn't subscribing to messages that are meant for another person. The
second ensures that the user is allowed to subscribe to any fields, by checking
a DynamoDB attribute of isAuthorisedForSubscriptions stored as a BOOL.

To summarise, the new newMessage subscription is triggered every time the
sendMessage mutation occurs. The newMessage resolvers query the Permissions
table for the message recipient and check a) if they have notifications
switched on, and b) if they are the actual recipient of the message before
returning the message data.

## VTL Resolver Mapping Templates

AppSync lets you respond to GraphQL operations by enabling you to perform
operations on your AWS resources. For each data source, a GraphQL resolver must
run and be able to communicate with that data source appropriately.

Usually, the communication is through parameters or operations that are unique
to the data source. For an AWS Lambda resolver, you need to specify the
payload. For an Amazon DynamoDB resolver, you need to specify a key. For an
Amazon Elasticsearch Service resolver, you need to specify an index and the
query operation.

Mapping templates are a way of indicating to AppSync how to translate an
incoming GraphQL request into instructions for your backend data source, and
how to translate the response from that data source back into a GraphQL
response. They are written in Apache Velocity Template Language (VTL), which
takes your request as input and outputs a JSON document containing the
instructions for the resolver. You can use mapping templates for simple
instructions, such as passing in arguments from GraphQL fields, or for more
complex instructions, such as looping through arguments to build an item before
inserting the item into DynamoDB.

There are two main types of mapping templates:

- Request templates:

  Take the incoming request after a GraphQL operation is parsed and convert it
  into instructions for the resolver so that the resolver can call your data
  source.

- Response templates:

  Interpret responses from your data source and translate into a GraphQL
  response, optionally performing some logic or formatting first.

See the [Resolver Mapping Template Reference](
https://docs.aws.amazon.com/appsync/latest/devguide/resolver-mapping-template-reference.html).

### Resolver Mapping Template Programming Guide

This is a cookbook-style tutorial of programming with the Apache Velocity
Template Language (VTL) in AWS AppSync.

AppSync uses VTL to translate GraphQL requests from clients into a request to
your data source. Then it reverses the process to translate the data source
response back into a GraphQL response. VTL is a logicful template language
that gives you the power to manipulate both the request and the response in
the standard request/response flow of a web application, using techniques such
as:

- Default values for new items
- Input validation and formatting
- Transforming and shaping data
- Iterating over lists, maps, and arrays to pluck out or alter values
- Filter/change responses based on user identity
- Complex authorisation checks

VTL allows you to apply logic using programming techniques that might be
familiar. However, it is bounded to run within the standard request/response
flow to ensure that your GraphQL API is scalable as your user base grows.

__As AppSync also supports Lambda as a resolver, you can always write Lambda
functions in your language of choice (Node.js, Python, Go, Java, etc.) if you
need more flexibility.__

#### Setup

Logging contents of resolver variables:

A common technique when learning a language is to print out results (for
example, console.log(variable) in JavaScript) to see what happens.

Create a Node.js Lambda function:

    exports.handler = (event, context, callback) => {
        console.log('VTL details: ', event);
        callback(null, event);
    };

Add this Lambda function as a the data source and resolver. See the results in
CloudWatch.

#### Variables

VTL uses references, which you can use to store or manipulate data. There are
three types of references in VTL: variables, properties, and methods.
Variables have a $ sign in front of them and are created with the #set
directive:

    #set($var = "a string")

Variables store types: numbers, strings, arrays, lists, and maps.

You might have noticed a JSON payload being sent in the default request
template for Lambda resolvers:

    "payload": $util.toJson($context.arguments)

A couple of things to notice here - first, AppSync provides several
convenience functions for common operations. In this example, $util.toJson
converts a variable to JSON. Second, the variable $context.arguments is
automatically populated from a GraphQL request as a map object. You can create
a new map as follows:

    #set( $myMap = {
      "id": $context.arguments.id,
      "meta": "stuff",
      "upperMeta" : $context.arguments.meta.toUpperCase()
    } )

Put this code at the top of your request template and change the payload to
use the new $myMap variable:

    "payload": $util.toJson($myMap)

You can also set properties on your variables. These could be simple strings,
arrays, or JSON:

    #set($myMap.myProperty = "ABC")
    #set($myMap.arrProperty = ["Write", "Some", "GraphQL"])
    #set($myMap.jsonProperty = {
        "AppSync" : "Offline and Realtime",
        "Cognito" : "AuthN and AuthZ"
    })

#### Quiet References

Because VTL is a templating language, by default, every reference you give it
will do a .toString(). If the reference is undefined, it prints the actual
reference representation, as a string. For example:

    #set($myValue = 5)
    ##Prints '5'
    $myValue

    ##Prints '$somethingelse'
    $somethingelse

To address this, VTL has a quiet reference or silent reference syntax, which
tells the template engine to suppress this behavior.
The syntax for this is $!{}.

    #set($myValue = 5)
    ##Prints '5'
    $myValue

    ##Nothing prints out
    $!{somethingelse}

#### Calling Methods

Above, we showed you how to create a variable and simultaneously set values.
You can also do this in two steps by adding data to your map as shown
following:

    #set ($myMap = {})
    #set ($myList = [])

    ##Nothing prints out
    $!{myMap.put("id", "first value")}
    ##Prints "first value"
    $!{myMap.put("id", "another value")}
    ##Prints true
    $!{myList.add("something")}

HOWEVER there is something to know about this behavior. Although the quiet
reference notation $!{} allows you to call methods, as above, it won't
suppress the returned value of the executed method. This is why we noted
Prints "first value" and Prints true above. This can cause errors when
you're iterating over maps or lists, such as inserting a value where a key
already exists, because the output adds unexpected strings to the template
upon evaluation.

The workaround to this is sometimes to call the methods using a #set directive
and ignore the variable. For example:

  #set ($myMap = {})
  #set($discard = $myMap.put("id", "first value"))

You might use this technique in your templates, as it prevents the unexpected
strings from being printed in the template. AppSync provides an alternative
convenience function that offers the same behavior in a more succinct
notation. This enables you to not have to think about these implementation
specifics. You can access this function under $util.quiet() or its alias
$util.qr(). For example:

    #set ($myMap = {})
    #set ($myList = [])

    ##Nothing prints out
    $util.quiet($myMap.put("id", "first value"))
    ##Nothing prints out
    $util.qr($myList.add("something"))

#### Strings

VTL quirks:

Suppose you are inserting data as a string to a data source like DynamoDB, but
it is populated from a variable, like a GraphQL argument. A string will have
double quotation marks, and to reference the variable in a string you just
need "${}" (so no ! as in quiet reference notation).

    #set($firstname = "Jeff")
    $!{myMap.put("Firstname", "${firstname}")}

You can see this in DynamoDB request templates, like "author": { "S" : "$
{context.arguments.author}"} when using arguments from GraphQL clients, or for
automatic ID generation like "id" : { "S" : "$utils.autoId()"}. This means
that you can reference a variable or the result of a method inside a string to
populate data.

You can also use public methods of the Java String class, such as pulling out
a substring:

    #set($bigstring = "This is a long string, I want to pull out everything after the comma")
    #set ($comma = $bigstring.indexOf(','))
    #set ($comma = $comma +2)
    #set ($substring = $bigstring.substring($comma))

    $util.qr($myMap.put("substring", "${substring}"))

String concatenation:

    #set($s1 = "Hello")
    #set($s2 = " World")
    $util.qr($myMap.put("concat","$s1$s2"))
    $util.qr($myMap.put("concat2","Second $s1 World"))

#### Loops

VTL allows only loops where the number of iterations is predetermined. There
is no do..while in Velocity. This design ensures that the evaluation process
always terminates, and provides bounds for scalability when your GraphQL
operations execute.

Loops are created with a #foreach and require you to supply a loop variable
and an iterable object such as an array, list, map, or collection. A classic
programming example with a #foreach loop is to loop over the items in a
collection and print them out:

    #set($start = 0)
    #set($end = 5)
    #set($range = [$start..$end])

    #foreach($i in $range)
      ##$util.qr($myMap.put($i, "abc"))
      ##$util.qr($myMap.put($i, $i.toString()+"foo")) ##Concat variable with string
      $util.qr($myMap.put($i, "${i}foo"))     ##Reference a variable in a string with "${varname}"
    #end

This example shows a few things. The first is using variables with the range
[..] operator to create an iterable object. Then each item is referenced by a
variable $i that you can operate with. In the previous example, you also see
Comments that are denoted with a double pound ##. This also showcases using
the loop variable in both the keys or the values, as well as different methods
of concatenation using strings.

You can also use a range operator directly, for example:

    #foreach($item in [1...5])
        ...
    #end

#### Arrays

With arrays you also have access to some underlying methods such as .isEmpty(),
.size(), .set(), .get(), and .add(), as shown below:

    #set($array = [])
    #set($idx = 0)

    ##adding elements
    $util.qr($array.add("element in array"))
    $util.qr($myMap.put("array", $array[$idx]))

    ##initialize array vals on create
    #set($arr2 = [42, "a string", 21, "test"])

    $util.qr($myMap.put("arr2", $arr2[$idx]))
    $util.qr($myMap.put("isEmpty", $array.isEmpty()))  ##isEmpty == false
    $util.qr($myMap.put("size", $array.size()))

    ##Get and set items in an array
    $util.qr($myMap.put("set", $array.set(0, 'changing array value')))
    $util.qr($myMap.put("get", $array.get(0)))

The previous example used array index notation to retrieve an element with
idx. You can look up by name from a Map/dictionary in a similar way:

    #set($result = {
        "Author" : "Nadia",
        "Topic" : "GraphQL"
    })

    $util.qr($myMap.put("Author", $result["Author"]))

This is very common when filtering results coming back from data sources in
Response Templates when using conditionals.

#### Conditional Checks

You can apply conditional checks to evaluate data at runtime:

    #if(!$array.isEmpty())
      $util.qr($myMap.put("ifCheck", "Array not empty"))
    #else
      $util.qr($myMap.put("ifCheck", "Your array is empty"))
    #end

The above #if() check of a Boolean expression is nice, but you can also use
operators and #elseif() for branching:

    #if ($arr2.size() == 0)
      $util.qr($myMap.put("elseIfCheck", "You forgot to put anything into this array!"))
    #elseif ($arr2.size() == 1)
      $util.qr($myMap.put("elseIfCheck", "Good start but please add more stuff"))
    #else
      $util.qr($myMap.put("elseIfCheck", "Good job!"))
    #end

These two examples showed negation(!) and equality (==).
We can also use ||, &&, >, <, >=, <=, and !=.

    #set($T = true)
    #set($F = false)

    #if ($T || $F)
      $util.qr($myMap.put("OR", "TRUE"))
    #end

    #if ($T && $F)
      $util.qr($myMap.put("AND", "TRUE"))
    #end
  
Note: Only Boolean.FALSE and null are considered false in conditionals. Zero
(0) and empty strings ("") are not equivalent to false.

#### Operators

    #set($x = 5)
    #set($y = 7)
    #set($z = $x + $y)
    #set($x-y = $x - $y)
    #set($xy = $x * $y)
    #set($xDIVy = $x / $y)
    #set($xMODy = $x % $y)

    $util.qr($myMap.put("z", $z))
    $util.qr($myMap.put("x-y", $x-y))
    $util.qr($myMap.put("x*y", $xy))
    $util.qr($myMap.put("x/y", $xDIVy))
    $util.qr($myMap.put("x|y", $xMODy))

#### Loops and Conditionals Together

It is very common when transforming data in VTL, such as before writing or
reading from a data source, to loop over objects and then perform checks
before performing an action. Combining some of the tools from the previous
sections gives you a lot of functionality. One handy tool is knowing that
foreach automatically provides you with a .count on each item:

    #foreach ($item in $arr2)
      #set($idx = "item" + $foreach.count)
      $util.qr($myMap.put($idx, $item))
    #end

For example, maybe you want to just pluck out values from a map if it is under
a certain size. Using the count along with conditionals and the #break
statement allows you to do this:

    #set($hashmap = {
      "DynamoDB" : "https://aws.amazon.com/dynamodb/",
      "Amplify" : "https://github.com/aws/aws-amplify",
      "DynamoDB2" : "https://aws.amazon.com/dynamodb/",
      "Amplify2" : "https://github.com/aws/aws-amplify"
    })

    #foreach ($key in $hashmap.keySet())
      #if($foreach.count > 2)
        #break
      #end
      $util.qr($myMap.put($key, $hashmap.get($key)))
    #end

The previous #foreach is iterated over with .keySet(), which you can use on
maps. This gives you access to get the $key and reference the value with a .get
($key). GraphQL arguments from clients in AWS AppSync are stored as a map.
They can also be iterated through with .entrySet(), which you can then access
both keys and values as a Set, and either populate other variables or perform
complex conditional checks, such as validation or transformation of input:

    #foreach( $entry in $context.arguments.entrySet() )
      #if ($entry.key == "XYZ" && $entry.value == "BAD")
        #set($myvar = "...")
      #else
        #break
      #end
    #end

Other common examples are autopopulating default information, like the initial
object versions when synchronizing data (very important in conflict resolution)
or the default owner of an object for authorisation checks - Mary created this
blog post, so:

    #set($myMap.owner ="Mary")
    #set($myMap.defaultOwners = ["Admins", "Editors"])

#### Context

Now that you are more familiar with performing logical checks in AppSync
resolvers with VTL, take a look at the context object:

    $util.qr($myMap.put("context", $context))

This contains all of the information that you can access in your GraphQL
request.

#### Filtering

$context.result is a map, so you can use entrySet() to perform logic on either
the keys or the values returned. Because $context.identity contains
information on the user that performed the GraphQL operation, if you return
authorisation information from the data source, then you can decide to return
all, partial, or no data to a user based on your logic. Change your response
template to look like the following:

    #if($context.result["id"] == 123)
        $utils.toJson($context.result);
      #else
        $util.unauthorised()
    #end

## Context map

Reference documentation: [Context map](
https://docs.aws.amazon.com/appsync/latest/devguide/resolver-context-reference.html)

### Accessing the $context map

The $context variable is a map which holds all of the contextual information for
your resolver invocation. It has the following structure:

    {
      "arguments" : { ... },
      "source" : { ... },
      "result" : { ... },
      "identity" : { ... }
    }

Note: If you are trying to access a dictionary/map entry (such as an entry in
context) by its key to retrieve the value, the VTL allows for you to directly
use the notation `<dictionary-element>.<key-name>`. However, this might not work
for all cases, such as when the key names have special characters (for example,
an underscore "_"). We recommend that you always use `<dictionary-element>.get
“<key-name>”`) notation.

Each field in the $context map is defined as follows:

- arguments
  
  A map containing all GraphQL arguments for this field.

- identity

  An object containing information about the caller.

- source

  A map containing the resolution of the parent field.

- result

  A map containing the results of this resolver. This map is only available to
  response mapping templates.

Example, if you are resolving the author field of the following query:

    query {
        getPost(id: 1234) {
            postId
            title
            content
            author {
                id
                name
            }
        }
    }

Then the full $context map might be:

    {
      "arguments" : {
        id: "1234"
      },
      "source": {},
      "result" : {
        "postId": "1234"
        "title": "Some title"
        "content": "Some content"
        "author": {
          "id": "5678"
          "name": "Author Name"
        }
      },
      "identity" : {
        "sourceIp" : ["x.x.x.x"],
        "userArn" : "arn:aws:iam::123456789012:user/appsync",
        "accountId" : "123456789012",
        "user" : "AIDAAAAAAAAAAAAAAAAAA"
      }
    }

### Identity

The identity section contains information about the caller. The shape of this
section depends on the authorisation type of your AWS AppSync API.

- API_KEY authorisation

  The identity field is not populated.

- AWS_IAM authorisation

  The identity has the following shape:

      {
          "accountId" = "string",
          "cognitoIdentityPoolId" = "string",
          "cognitoIdentityId" = "string",
          "sourceIp" = ["string"],
          "username" = "string", // IAM user principal
          "userArn" = "string"
      }

- AMAZON_COGNITO_USER_POOLS authorisation

  The identity has the following shape:

      {
          "sub" : "uuid",
          "issuer" : "string",
          "username" : "string"
          "claims" : { ... },
          "sourceIp" : ["x.x.x.x"],
          "defaultAuthStrategy" : "string"
      }

Each field is defined as follows:

- accountId

    The AWS account ID of the caller.

- claims

    The claims the user has.

- cognitoIdentityId

    The Amazon Cognito identity ID of the caller.

- cognitoIdentityPoolId

    The Amazon Cognito identity pool ID associated with the caller.

- defaultAuthStrategy

    The default auth strategy for this caller (ALLOW or DENY).

- issuer

    The token issuer.

- sourceIp

    The source IP address of the caller received by AppSync. If the request
    doesn't include the x-forwarded-for header, the source IP value contains
    only a single IP Address from the TCP Connection. If the request includes a
    x-forwarded-for header, the source IP is a list of IP Addresses from the
    x-forwarded-for header, in addition to the IP address from the TCP
    connection.

- sub

    The UUID of the authenticated user.

- user

    The IAM user.

- userArn

    The IAM user ARN.

- username

    The username of the authenticated user. In case of AMAZON_COGNITO_USER_POOLS
    authorisation, the value of username is the value of attribute
    cognito:username. In case of AWS_IAM authorisation, the value of the
    username is the value of the AWS User Principal. We recommend that you use
    cognitoIdentityId if you are using AWS IAM authorisation with credentials
    vended from Amazon Cognito federated identities.

### Access Request Headers

AppSync supports passing custom headers from clients and accessing them in your
GraphQL resolvers using $context.request.headers. You can then use the header
values for actions like inserting data to a data source or even authorisation
checks. Single or multiple request headers can ne used as shown in the
following examples using $curl with an API key from the command line:

#### Single Header Example

Suppose you set a header of custom with a value of nadia like so:

    curl -XPOST \
      -H "Content-Type:application/graphql" \
      -H "custom:nadia" \
      -H "x-api-key:<API-KEY-VALUE>" \
      -d '{"query":"mutation { createEvent(name: \"demo\", when: \"Next Friday!\", where: \"Here!\") {id name when where description}}"}' \
      https://<ENDPOINT>/graphql

This could then be accessed with $context.request.headers.custom. For example,
it might be in the following VTL for DynamoDB:

    "custom": { "S": "$context.request.headers.custom" }

#### Multiple Header Example

You can also pass multiple headers in a single request and access these in the
resolver mapping template. For example, if the custom header was set with two
values:

    curl -XPOST \
      -H "Content-Type:application/graphql" \
      -H "custom:bailey" \
      -H "custom:nadia" \
      -H "x-api-key:<API-KEY-VALUE>" \
      -d '{"query":"mutation { createEvent(name: \"demo\", when: \"Next Friday!\", where: \"Here!\") {id name when where description}}"}' \
      https://<ENDPOINT>/graphql

You could then access these as an array, such as

    $context.request.headers.custom[1]

## Fingerprinting AppSync files on S3

AppSync resolvers and the GraphQL schema can be stored on S3. This makes the
CloudFormation template more manageable. The .vtl resolver files will also be
easier to read with the correct syntax highlighting provided by this vscode
[this extension](https://github.com/luqimin/tinyvm).

Fingerprinting is handled by the AWS CLI [package](
https://docs.aws.amazon.com/cli/latest/reference/cloudformation/package.html) 
command, for example:

    aws cloudformation package \
      --template-file ./appsync-author.yaml  \
      --output-template-file ./appsync-author.pkg.yaml \
      --s3-bucket mybucket \
      --s3-prefix infrastructure/package \
      --region eu-west-1

This command packages the local artifacts (local paths) that the CloudFormation
template references. The command uploads the local artifacts, such as resolver
files, to an S3 bucket. The command returns a copy of the template, replacing
references to local artifacts with the fingerprinted S3 location where the
command uploaded the artifacts.

## Troubleshooting and Common Mistakes

### Incorrect DynamoDB Key Mapping

If your GraphQL operation returns the following error message, it may be because
your request mapping template structure doesn't match the DynamoDB key structure:

    The provided key element does not match the schema (Service: AmazonDynamoDBv2; Status Code: 400; Error Code

For example, if your DynamoDB table has a hash key called "id" and your template
says "PostID", as in the following example, this results in the preceding error,
because "id" doesn't match "PostID".

    {
      "version" : "2017-02-28",
      "operation" : "GetItem",
      "key" : {
        "PostID" : { "S" : "${context.arguments.id}" }
      }
    }

### Missing Resolver

If you execute a GraphQL operation, such as a query, and get a null response,
this may be because you don't have a resolver configured.

For example, if you import a schema that defines a getCustomer(userId: ID!):
field, and you haven't configured a resolver for this field, then when you
execute a query such as:

    getCustomer(userId:"ID123"){...}

you'll get a response such as the following:

    {
      "data": {
        "getCustomer": null
      }
    }

### Mapping Template Errors

If your mapping template isn't properly configured, you'll receive a GraphQL
response whose errorType is MappingTemplate. The message field should indicate
where the problem is in your mapping template.

For example, if you don't have an operation field in your request mapping
template, or if the operation field name is incorrect, you'll get a response
like the following:

    {
      "data": {
        "searchPosts": null
      },
      "errors": [
        {
        "path": [
          "searchPosts"
        ],
        "errorType": "MappingTemplate",
        "locations": [
          {
          "line": 2,
          "column": 3
          }
        ],
        "message": "Value for field '$[operation]' not found."
        }
      ]
    }

### Incorrect Return Types

The return type from your data source must match the defined type of an object
in your schema, otherwise you may see a GraphQL error like:

    "errors": [
      {
        "path": [
          "posts"
        ],
        "locations": null,
        "message": "Can't resolve value (/posts) : type mismatch error, expected
        type LIST, got OBJECT"
      }
    ]

For example this could occur with the following query definition:

    type Query {
      posts: [Post]
    }

Which expects a LIST of [Posts] objects. For example if you had a Lambda
function in Node.JS with something like the following:

    const result = { data: data.Items.map(item => { return item ; }) };
    callback(err, result);

This would throw an error as result is an object. You would need to either
change the callback to result.data or alter your schema to not return a LIST.

## FAQ

- How do you programmatically test a GraphQL API on AppSync?

  Testing REST and GraphQL Services With Just-API:

  <https://dzone.com/articles/testing-rest-graphql-services>

  You can mock the backend for your GraphQL schema locally:

  <https://graphql.org/blog/mocking-with-graphql>

  Although this won't help with AppSync resolvers.

- How to curl an AppSync API:
  [source](https://sbstjn.com/aws-appsync-graphql-with-cloudformation.html)

      curl \
          -XPOST https://appsync.example.com/graphql \
          -H "Content-Type:application/graphql" \
          -H "x-api-key:da2-nuyzhanm5bcptga6yilkj7zluy" \
          -d '{ "query": "query { feed { title url } }" }' | jq

      {
        "data": {
          "feed": [
            {
              "title": "Deploy Golang Lambda with AWS Serverless Application Model",
              "url": "https://sbstjn.com/golang-lambda-with-aws-sam-serverless-application-model.html"
            },
            {
              "title": "68% Mechanical Keyboard with 68Keys.io",
              "url": "https://sbstjn.com/build-your-own-mechanical-keyboard.html"
            }
            [...]
        }
      }