# GraphQL

## Contents

- [Resources](#Resources)
- [Summary](#Summary)
- [GraphQL Clients](#GraphQL-Clients)

## Resources

- [GraphQL.org](https://graphql.org)
- [Tutorials](https://www.howtographql.com)

## Summary

[Source](
https://docs.aws.amazon.com/appsync/latest/devguide/graphql-overview.html)

GraphQL is a data language that was developed to enable apps to fetch data
from servers. It has a declarative, self-documenting style. In a GraphQL
operation, the client specifies how to structure the data when it is returned
by the server. This makes it possible for the client to query only for the
data it needs, in the format that it needs it in.

GraphQL has three top-level operations:

- Query - read-only fetch
- Mutation - write, followed by a fetch
- Subscription - long-lived connection for receiving data (mutations can
  invoke one or more subscriptions)

GraphQL exposes these operations via a schema that defines the capabilities of
an API. A schema is comprised of types, which can be root types (query,
mutation, or subscription) or user-defined types. Developers start with a
schema to define the capabilities of their GraphQL API, which a client
application will communicate with. Any field that ends in an exclamation point
is a required field.

After a schema is defined, the fields on a type need to return some data. The
way this happens in a GraphQL API is through a GraphQL resolver. This is a
function that either calls out to a data source or invokes a trigger to return
some value (such as an individual record or a list of records).

### Relationships

    type Comment {
        todoid: ID!
        commentid: String!
        content: String
    }

The Comment type has the todoid that it's associated with, commentid, and
content. This corresponds to a primary key + sort key combination in the
Amazon DynamoDB table you create later.

### Interfaces

GraphQL's type system features [Interfaces](
https://graphql.org/learn/schema/#interfaces).
An interface exposes a certain set of fields that a type must include to
implement the interface. Using interfaces to represent common characteristics is very helpful for sorting results.

### Unions

GraphQL's type system also features Unions. Unions are identical to interfaces,
except that they don't define a common set of fields. Unions are generally
preferred over interfaces when the possible types do not share a logical
hierarchy. These are often used for searches.

Hints are required to identify the type of the search result as this can
depend on the app.

E.g. with AWS AppSync, this hint is represented by a meta field named
__typename, whose value corresponds to the identified object type name.
__typename is required for return types that are interfaces or unions.

You can put some logic in the response resolver (or in a lambda function if
you are using the Lambda data source) to work out the __typename:

    #foreach ($result in $context.result)
        ## Extract type name from the id field.
        #set( $typeName = $result.id.split("-")[0] )
        #set( $ignore = $result.put("__typename", $typeName))
    #end
    $util.toJson($context.result)

In this example the typeName is extracted from the id in the query results.

## GraphQL Clients

- [Apollo GraphQL Client](https://github.com/apollographql/apollo-client)
- [Ember addon for Apollo Client and GraphQL](
  https://github.com/bgentry/ember-apollo-client)
- [AWS Amplify](
  https://aws-amplify.github.io/amplify-js/media/api_guide#working-with-graphql)
