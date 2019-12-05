![Serverless AppSync Component](https://s3.amazonaws.com/assets.general.serverless.com/component_appsync/readme-appsync-serverless-component.gif)

# Serverless AppSync Component

The AppSync [Serverless Component](https://github.com/serverless/components) allows you to easily and quickly deploy GraphQL APIs on AWS, and integrate them with AWS Lambda, DynamoDB & others. It supports all AWS AppSync features, while offering sane defaults that makes working with AppSync a lot easier without compromising on flexibility.

## Features

- [x] Fast Deployments (~10 seconds on average)
- [x] Create New APIs or Reuse Existing Ones
- [x] Supports Custom Domains with CDN & SSL Out of the Box
- [x] Supports Custom AppSync Service Role
- [x] Supports Lambda Data Source
- [x] Supports DynamoDB Data Source
- [x] Supports ElasticSearch Data Source
- [x] Supports Relational Database Data Source
- [x] Supports API Keys Authentication
- [x] Supports Cognito User Pools Authentication
- [x] Supports OpenID Connect Authentication
- [x] Supports AppSync Functions

## Contents

1. [Install](#1-install)
2. [Create](#2-create)
3. [Configure](#3-configure)
   - [Basic Configuration](#basic-configuration)
   - [Custom Domain](#custom-domains)
   - [Create or Reuse APIs](#create-or-reuse-apis)
   - [Authentication](#authentication)
   - [Schema Configuration](#schema)
   - [Data Sources & Templates](#data-sources--templates)
   - [Functions](#functions)
4. [Deploy](#4-deploy)

## 1. Install

```shell
$ npm install -g serverless
```

## 2. Create

Just create the following simple boilerplate:

```
$ touch serverless.yml # more info in the "Configure" section below
$ touch schema.graphql # your graphql schema file
$ touch index.js       # only required if you use a Lambda data source
$ touch .env           # your AWS api keys
```

```
# .env
AWS_ACCESS_KEY_ID=XXX
AWS_SECRET_ACCESS_KEY=XXX
```

## 3. Configure

### Basic Configuration
The following is a simple configuration that lets you get up and running quickly with a Lambda data source. Just add it to the `serverless.yml` file:


```yml
myLambda:
  component: "@serverless/aws-lambda"
  inputs:
    handler: index.handler
    code: ./

myAppSyncApi:
  component: "@serverless/aws-app-sync"
  inputs:
    # creating the API and an API key
    name: Posts
    authenticationType: API_KEY
    apiKeys:
      - myApiKey

    # defining your lambda data source
    dataSources:
      - type: AWS_LAMBDA
        name: getPost
        config:
          lambdaFunctionArn: ${myLambda.arn}

    # mapping schema fields to the data source
    mappingTemplates:
      - dataSource: getPost
        type: Query
        field: getPost
```
This configuration works with the following example schema. Just add it to the `schema.graphql` file right next to `serverless.yml`:

```graphql
schema {
  query: Query
}

type Query {
  getPost(id: ID!): Post
}

type Post {
  id: ID!
  author: String!
  title: String
  content: String
  url: String
}
```
You'll also need to add the following handler code for this example to work:

```js
exports.handler = async event => {
  var posts = {
    "1": {
      id: "1",
      title: "First Blog Post",
      author: "Eetu Tuomala",
      url: "https://serverless.com/",
      content:
        "Lorem Ipsum is simply dummy text of the printing and typesetting industry. Lorem Ipsum has been the industry's standard dummy text ever since the 1500s."
    },
    "2": {
      id: "2",
      title: "Second Blog Post",
      author: "Siddharth Gupta",
      url: "https://serverless.com",
      content:
        "Lorem Ipsum is simply dummy text of the printing and typesetting industry. Lorem Ipsum has been the industry's standard dummy text ever since the 1500s."
    }
  };

  return posts[event.id];
};
```

For more advanced usage, keep reading!

### Custom Domains
You could optionally specify a custom domain for your GraphQL API, just add a domain property to the app sync component inputs:

```yml
myAppSyncApi:
  component: "@serverless/aws-app-sync"
  inputs:
    domain: api.example.com # add your custom domain here
    name: Posts
    # ... rest of config here

```
This would create a CloudFront distribution (aka CDN) for your AppSync API, which reduces request latency significantly, and would give you an SSL certificate out of the box powered by AWS ACM.

Please note that your domain (example.com in this example) must have been purchased via AWS Route53 and available in your AWS account. For advanced users, you may also purchase it elsewhere, then configure the name servers to point to an AWS Route53 hosted zone. How you do that depends on your registrar.

### Create or Reuse APIs

The AppSync component allows you to either create an AppSync API from scratch, or integrate with an existing one. Here's how to create a new API:

```yml
# serverless.yml

myAppSync:
  component: '@serverless/aws-app-sync'
  inputs:
    name: 'my-api-name' # specifying a name creates a new API
    domain: api.example.com # example.com must be available in your AWS Route53
    authenticationType: 'API_KEY'
    apiKeys:
      - 'myApiKey'
    mappingTemplates:
      - dataSource: 'dynamodb_ds'
        type: 'Query'
        field: 'getMyField'
        request: 'mapping-template-request.vtl'
        response: 'mapping-template-response.vtl'
    functions:
      - dataSource: 'dynamodb_ds'
        name: 'my-function'
        request: 'function-request.vtl'
        response: 'function-response.vtl'
    dataSources:
      - type: 'AMAZON_DYNAMODB'
        name: 'dynamodb_ds'
        config:
          tableName: 'my-dynamo-table'
    schema: 'schema.graphql' # optional. Default behavior would look for schema.graphql file in the cwd
```

Reuse an existing AWS AppSync service by adding replacing the `name` input with `apiId`. This way the component will modify the AppSync service by those parts which are defined.

```yml
# serverless.yml

myAppSync:
  component: '@serverless/aws-app-sync'
  inputs:
    apiId: 'm3vv766ahnd6zgjofnri5djnmq'
    mappingTemplates:
      - dataSource: 'dynamodb_2_ds'
        type: 'Query'
        field: 'getMyField'
        request: 'mapping-template-request.vtl'
        response: 'mapping-template-response.vtl'
    dataSources:
      - type: 'AMAZON_DYNAMODB'
        name: 'dynamodb_2_ds'
        config:
          tableName: 'my-dynamo-table'
```

### Authentication

The app using AppSync API can use four different methods for authentication.

- API_KEY - Api keys
- AWS_IAM - IAM Permissions
- OPENID_CONNECT - OpenID Connect provider
- AMAZON_COGNITO_USER_POOLS - Amazon Cognito user pool

When using OpenID connect method, inputs has to contain `openIDConnectConfig` block.

```yaml
myAppSync:
  component: '@serverless/aws-app-sync'
  inputs:
    authenticationType: 'OPENID_CONNECT'
    openIDConnectConfig:
      issuer: 'NNN'
      authTTL: '1234'
      clientId: 'NNN'
      iatTTL: '1234'
```

When using Amazon Cognito user pools, `userPoolConfig` has to be defined.

```yaml
myAppSync:
  component: '@serverless/aws-app-sync'
  inputs:
    authenticationType: 'AMAZON_COGNITO_USER_POOLS'
    userPoolConfig:
      awsRegion: 'us-east-1'
      defaultAction: 'ALLOW'
      userPoolId: 'us-east-1_nnn'
```

ApiKey can be created and modified by defining `apiKeys`.

```yaml
myAppSync:
  component: '@serverless/aws-app-sync'
  inputs:
    apiKeys:
      - 'myApiKey1' # using default expiration data
      - name: 'myApiKey2'
        expires: 1609372800
      - name: 'myApiKey3'
        expires: '2020-12-31'
```

### Schema
You can define the schema of your GraphQL API by adding it to the `schema.graphql` file right next to `serverless.yml`. Here's a simple example schema:

```graphql
schema {
  query: Query
}

type Query {
  getPost(id: ID!): Post
}

type Post {
  id: ID!
  author: String!
  title: String
  content: String
  url: String
}
```

Alternatively, if you have your schema file at a different location, you can specify the new location in `serverless.yml`

```yml
  inputs:
    name: myGraphqlApi
    schema: ./path/to/schema.graphql # specify your schema location
```

### Data Sources & Templates
The AppSync component supports 4 AppSync data sources and their corresponding mapping templates. You could add as many data sources as your application needs. For each field (or operation) in your Schema (ie. `getPost`), you'll need to add a mapping template that maps to a data source.

Here are the data sources that are supported:

#### Lambda Data Source
Here's an example setup for a Lambda data source:

**serverless.yml**

```yml
myLambda:
  component: "@serverless/aws-lambda"
  inputs:
    handler: index.handler
    code: ./

myAppSyncApi:
  component: "@serverless/aws-app-sync"
  inputs:
    # creating the API and an API key
    name: Posts
    authenticationType: API_KEY
    apiKeys:
      - myApiKey

    # defining your lambda data source
    dataSources:
      - type: AWS_LAMBDA
        name: getPost
        config:
          lambdaFunctionArn: ${myLambda.arn} # pass the lambda arn from the aws-lambda component above
      - type: AWS_LAMBDA
        name: addPost
        config:
          lambdaFunctionArn: ${myLambda.arn} # you could pass another lambda ARN, or the same one if it handles that field

    # mapping schema fields to the data source
    mappingTemplates:
      - dataSource: getPost
        type: Query
        field: getPost
      - dataSource: addPost
        type: Mutation
        field: addPost

        # Minimal request/response templates are added by default that works for 99% of use cases.
        # But you could also overwrite them with your own templates by specifying the path to the template files
        request: request.vtl
        response: response.vtl
```

**index.js**

```js
exports.handler = async (event) => {
  var posts = {
    '1': {
      id: '1',
      title: 'First Blog Post',
      author: 'Eetu Tuomala',
      url: 'https://serverless.com/',
      content:
        "Lorem Ipsum is simply dummy text of the printing and typesetting industry. Lorem Ipsum has been the industry's standard dummy text ever since the 1500s."
    },
    '2': {
      id: '2',
      title: 'Second Blog Post',
      author: 'Siddharth Gupta',
      url: 'https://serverless.com',
      content:
        "Lorem Ipsum is simply dummy text of the printing and typesetting industry. Lorem Ipsum has been the industry's standard dummy text ever since the 1500s."
    }
  }

  return posts[event.id]
}


```

**schema.graphql**

```graphql
schema {
  query: Query
}

type Query {
  getPost(id: ID!): Post
}

type Post {
  id: ID!
  author: String!
  title: String
  content: String
  url: String
}

```

#### DynamoDB Data Source
For the DynamoDB data source, you'll need to provide your own request/response templates that works according to your schema. Here's an example setup:

**serverless.yml**

```yml

# create a table with the aws-dynamodb component
myTable:
  component: '@serverless/aws-dynamodb'

appsync:
  component: '@serverless/aws-app-sync'
  inputs:
    name: Posts
    authenticationType: API_KEY
    apiKeys:
      - myApiKey
    dataSources:
      - type: AMAZON_DYNAMODB
        name: Posts
        config:
          tableName: ${myTable.name}
    mappingTemplates:
      - dataSource: Posts
        type: Mutation
        field: addPost
        request: request.vtl
        response: response.vtl
```

**request.vtl**

```
{
    "version" : "2017-02-28",
    "operation" : "PutItem",
    "key" : {
        "id" : { "S" : "${context.arguments.id}" }
    },
    "attributeValues" : {
        "author": { "S" : "${context.arguments.author}" },
        "title": { "S" : "${context.arguments.title}" },
        "content": { "S" : "${context.arguments.content}" },
        "url": { "S" : "${context.arguments.url}" },
        "ups" : { "N" : 1 },
        "downs" : { "N" : 0 },
        "version" : { "N" : 1 }
    }
}
```

**response.vtl**
```
$util.toJson($context.result)
```

**schema.graphql**

```graphql
schema {
  query: Query
  mutation: Mutation
}

type Query {
  getPost(id: ID): Post
}

type Mutation {
  addPost(id: ID!, author: String!, title: String!, content: String!, url: String!): Post!
}

type Post {
  id: ID!
  author: String
  title: String
  content: String
  url: String
  ups: Int!
  downs: Int!
  version: Int!
}

```

#### ElasticSearch Data Source

This example uses Amazon ElasticSearch Service running in AWS. The cluster contains ElasticSearch/Kibana sample flight data https://www.elastic.co/guide/en/kibana/current/tutorial-sample-data.html.

**serverless.yml**

```yaml
myAppSyncApi:
  component: "@serverless/aws-app-sync"
  inputs:
    # creating the API and an API key
    name: Flights
    authenticationType: API_KEY
    apiKeys:
      - myApiKey

    dataSources:
      - type: AMAZON_ELASTICSEARCH
        name: Flights
        config:
          endpoint: https://search-my-sample-data-abbaabba.us-east-1.es.amazonaws.com

    mappingTemplates:
      - dataSource: Flights
        type: Query
        field: getFlights
        request: request.vtl
        response: response.vtl
```

**request.vtl**

The resolver returns 50 latest flights.

```
{
    "version":"2017-02-28",
    "operation":"GET",
    "path":"/kibana_sample_data_flights/_doc/_search",
    "params":{
        "body": {
            "from": 0,
            "size": 50,
            "query": {
                "match_all": {}
            }
        }
    }
}
```

**response.vtl**

```
[
    #foreach($entry in $context.result.hits.hits)
        ## $velocityCount starts at 1 and increments with the #foreach loop **
        #if( $velocityCount > 1 ) , #end
        $util.toJson($entry.get("_source"))
    #end
]
```

**schema.graphql**

```graphql
type Flight {
	FlightNum: String
	DestAirportID: String
	OriginAirportID: String
}

type Query {
	getFlights: [Flight]
}

schema {
	query: Query
}
```

After deployment, update ElasticSearch access policy to allow created role by adding following block to statement array.

```json
{
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::123456789012:role/created-role" <- replace with correct role
  },
  "Action": [
    "es:ESHttpDelete",
    "es:ESHttpHead",
    "es:ESHttpGet",
    "es:ESHttpPost",
    "es:ESHttpPut"
  ],
  "Resource": "arn:aws:es:us-east-1:123456789012:domain/my-sample-data/*" <- replace with correct ElasticSearch domain
}
```

#### Relational Database Data Source

This example is using an Amazon Aurora Serverless cluster with PostgreSQL database which is already running in AWS.

The database has two tables, `authors` table, which contains an `id` and a `fullname` columns, and `posts` table which contains following columns `id`, `title`, `content`, `url`, and a foreign key `fk_authors_id`.

**serverless.yml**

```yaml
myAppSyncApi:
  component: "@serverless/aws-app-sync"
  inputs:
    # creating the API and an API key
    name: Posts
    authenticationType: API_KEY
    apiKeys:
      - myApiKey

    # Relational database datasource has to be an Amazon Aurora Serverless Cluster with Data API enabled
    # https://docs.aws.amazon.com/appsync/latest/devguide/tutorial-rds-resolvers.html
    dataSources:
      - type: RELATIONAL_DATABASE
        name: Posts
        config:
          awsSecretStoreArn: 'arn:aws:secretsmanager:us-east-1:123456789123:secret:rds-db-credentials/cluster-ABCDEFGHI/admin-aBc1e2'
          databaseName: 'mydatabase'
          dbClusterIdentifier: 'my-serverless-aurora-postgres-1'
          schema: 'public'

    mappingTemplates:
      - dataSource: Posts
        type: Query
        field: getPost
        request: request.vtl
        response: response.vtl
```

**request.vtl**

```
{
    "version": "2018-05-29",
    "statements": [
        "select posts.id, posts.title, posts.content, posts.url, authors.fullname as author from posts, authors where authors.id = posts.fk_authors_id and posts.id = '$ctx.args.id'"
    ]
}
```

**response.vtl**
```
#if($ctx.error)
    $utils.error($ctx.error.message, $ctx.error.type)
#end

$utils.toJson($utils.rds.toJsonObject($ctx.result)[0][0])
```

**schema.graphql**

```graphql
schema {
  query: Query
  mutation: Mutation
}

type Query {
  getPost(id: ID): Post
}

type Mutation {
  addPost(fk_authors_id: Int!, title: String!, content: String!, url: String!): Post!
}

type Post {
  id: ID!
  author: String
  fk_authors_id: Int
  title: String
  content: String
  url: String
}
```

### Functions

## 4. Deploy
To deploy, just run the following command in the directory containing your `serverless.yml file`:

```shell
$ serverless
```

After few seconds (up to a minute if it's your first deployment), you should see an output like this:

```
  myAppSyncApi:
    apiId:   samrhyo7srbtvkpqnj4j6uq6gq
    arn:     arn:aws:appsync:us-east-1:552751238299:apis/samrhyo7srbtvkpqnj4j6uq6gq
    url:     "https://samrhyo7srbtvkpqnj4j6uq6gq.appsync-api.us-east-1.amazonaws.com/graphql"
    apiKeys:
      - da2-coeytoubhffnfastengavajsku
    domain:  "https://api.example.com/graphql"

  9s › myAppSyncApi › done

myApp (master)$
```

&nbsp;

## New to Components?

Checkout the [Serverless Components](https://github.com/serverless/components) repo for more information.
