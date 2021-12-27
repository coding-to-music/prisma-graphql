# How To Build a GraphQL API with Prisma and Deploy to DigitalOcean's App Platform

https://www.digitalocean.com/community/tutorials/how-to-build-a-graphql-api-with-prisma-and-deploy-to-digitalocean-s-app-platform

Node.js Git APIDevelopment GraphQL DigitalOcean App Platform

danielnorman

By Daniel Norman https://www.digitalocean.com/community/users/danielnorman

Published onOctober 28, 2020 16.9kviews
The author selected the COVID-19 Relief Fund to receive a donation as part of the Write for DOnations program.

## Introduction
GraphQL is a query language for APIs that consists of a schema definition language and a query language, which allows API consumers to fetch only the data they need to support flexible querying. GraphQL enables developers to evolve the API while meeting the different needs of multiple clients, for example iOS, Android, and web variants of an app. Moreover, the GraphQL schema adds a degree of type safety to the API while also serving as a form of documentation for your API.

Prisma is an open-source database toolkit. It consists of three main tools:

- Prisma Client: Auto-generated and type-safe query builder for Node.js & TypeScript.
- Prisma Migrate: Declarative data modeling & migration system.
- Prisma Studio: GUI to view and edit data in your database.
- Prisma facilitates working with databases for application developers who want to focus on implementing value-adding features instead of spending time on complex database workflows (such as schema migrations or writing complicated SQL queries).

In this tutorial, you will use GraphQL and Prisma in combination as their responsibilities complement each other. GraphQL provides a flexible interface to your data for use in clients, such as frontends and mobile apps—GraphQL isn’t tied to any specific database. This is where Prisma comes in to handle the interaction with the database where your data will be stored.

DigitalOcean’s App Platform provides a seamless way to deploy applications and provision databases in the cloud without worrying about infrastructure. This reduces the operational overhead of running an application in the cloud; especially with the ability to create a managed PostgreSQL database with daily backups and automated failover. App Platform has native Node.js support streamlining deployment.

You’ll build a GraphQL API for a blogging application in JavaScript using Node.js. You will first use Apollo Server to build the GraphQL API backed by in-memory data structures. You will then deploy the API to the DigitalOcean App Platform. Finally you will use Prisma to replace the in-memory storage and persist the data in a PostgreSQL database and deploy the application again.

At the end of the tutorial, you will have a Node.js GraphQL API deployed to DigitalOcean, which handles GraphQL requests sent over HTTP and performs CRUD operations against the PostgreSQL database.

You can find the code for this project in the DigitalOcean Community respository.

## Prerequisites
Before you begin this guide you’ll need the following:

- A GitHub account.
- A DigitalOcean account.
- Git installed on your computer. You can follow the tutorial Contributing to Open Source: Getting Started with - Git to install and set up Git on your computer.
- Node.js v10 or higher installed on your computer. You can follow the tutorial How to Install Node.js and Create a Local Development Environment to install and set up Node.js on your computer.
- Docker installed on your computer (to run the PostgreSQL database locally).
- Basic familiarity with JavaScript, Node.js, GraphQL, and PostgreSQL is helpful, but not strictly required for this tutorial.

## Step 1 — Creating the Node.js Project
In this step, you will set up a Node.js project with npm and install the dependencies apollo-server and graphql.

This project will be the foundation for the GraphQL API that you’ll build and deploy throughout this tutorial.

First, create a new directory for your project:

```java
mkdir prisma-graphql
```
 
Next, navigate into the directory and initialize an empty npm project:

```java
cd prisma-graphql
npm init --yes
```
 
This command creates a minimal package.json file that is used as the configuration file for your npm project.

You will receive the following output:

Output
```java
Wrote to /Users/yourusaername/workspace/prisma-graphql/package.json:
{
  "name": "prisma-graphql",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC"
}
```

You’re now ready to configure TypeScript in your project.

Execute the following command to install the necessary dependencies:

```java
npm install apollo-server graphql --save
```
 
This installs two packages as dependencies in your project:

- apollo-server: The HTTP library that you use to define how GraphQL requests are resolved and how to fetch data.
- graphql: is the library you’ll use to build the GraphQL schema.
You’ve created your project and installed the dependencies. In the next step you will define the GraphQL schema.

## Step 2 — Defining the GraphQL Schema and Resolvers
In this step, you will define the GraphQL schema and corresponding resolvers. The schema will define the operations that the API can handle. The resolvers will define the logic for handling those requests using in-memory data structures, which you will replace with database queries in the next step.

First, create a new directory called src that will contain your source files:

```java
mkdir src
```
 
Then run the following command to create the file for the schema:

```java
nano src/schema.js
```
 
Now add the following code to the file:


prisma-graphql/src/schema.js
```java
const { gql } = require('apollo-server')

const typeDefs = gql`
  type Post {
    content: String
    id: ID!
    published: Boolean!
    title: String!
  }

  type Query {
    feed: [Post!]!
    post(id: ID!): Post
  }

  type Mutation {
    createDraft(content: String, title: String!): Post!
    publish(id: ID!): Post
  }
`
```
 
Here you define the GraphQL schema using the gql tagged template. A schema is a collection of type definitions (hence typeDefs) that together define the shape of queries that can be executed against your API. This will convert the GraphQL schema string into the format that Apollo expects.

The schema introduces three types:

- Post: Defines the type for a post in your blogging app and contains four fields where each field is followed by its type, for example, String.
- Query: Defines the feed query which returns multiple posts as denoted by the square brackets and the post query which accepts a single argument and returns a single Post.
- Mutation: Defines the createDraft mutation for creating a draft Post and the publish mutation which accepts an id and returns a Post.
- Note that every GraphQL API has a query type and may or may not have a mutation type. These types are the same as a regular object type, but they are special because they define the entry point of every GraphQL query.

Next, add the posts array to the src/schema.js file, below the typeDefs variable:

prisma-graphql/src/schema.js
```java
...
const posts = [
  {
    id: 1,
    title: 'Subscribe to GraphQL Weekly for community news ',
    content: 'https://graphqlweekly.com/',
    published: true,
  },
  {
    id: 2,
    title: 'Follow DigitalOcean on Twitter',
    content: 'https://twitter.com/digitalocean',
    published: true,
  },
  {
    id: 3,
    title: 'What is GraphQL?',
    content: 'GraphQL is a query language for APIs',
    published: false,
  },
]
```
 
You define the posts array with three pre-defined posts. Notice that the structure of each post object matches the Post type you defined in the schema. This array holds the posts that will be served by the API. In a subsequent step, you will replace the array once the database and Prisma Client are introduced.

Next, define the resolvers object below the posts array you just defined:

prisma-graphql/src/schema.js
```java
...
const resolvers = {
  Query: {
    feed: (parent, args) => {
      return posts.filter((post) => post.published)
    },
    post: (parent, args) => {
      return posts.find((post) => post.id === Number(args.id))
    },
  },
  Mutation: {
    createDraft: (parent, args) => {
      posts.push({
        id: posts.length + 1,
        title: args.title,
        content: args.content,
        published: false,
      })
      return posts[posts.length - 1]
    },
    publish: (parent, args) => {
      const postToPublish = posts.find((post) => post.id === Number(args.id))
      postToPublish.published = true
      return postToPublish
    },
  },
  Post: {
    content: (parent) => parent.content,
    id: (parent) => parent.id,
    published: (parent) => parent.published,
    title: (parent) => parent.title,
  },
}


module.exports = {
  resolvers,
  typeDefs,
}
```
 
You define the resolvers following the same structure as the GraphQL schema. Every field in the schema’s types has a corresponding resolver function whose responsibility is to return the data for that field in your schema. For example, the Query.feed() resolver will return the published posts by filtering the posts array.

Resolver functions receive four arguments:

- parent: The parent is the return value of the previous resolver in the resolver chain. For top-level resolvers, the parent is undefined, because no previous resolver is called. For example, when making a feed query, the query.feed() resolver will be called with parent’s value undefined and then the resolvers of Post will be called where parent is the object returned from the feed resolver.
- args: This argument carries the parameters for the query, for example, the post query, will receive the id of the post to be fetched.
- context: An object that gets passed through the resolver chain that each resolver can write to and read from, which allows the resolvers to share information.
- info: An AST representation of the query or mutation. You can read more about the details in part III of this series: Demystifying the info Argument in GraphQL Resolvers.
- Since the context and info are not necessary in these resolvers, only parent and args are defined.

Save and exit the file once you’re done.

Note: When a resolver returns the same field as the resolver’s name, like the four resolvers for Post, Apollo Server will automatically resolve those. This means you don’t have to explicitly define those resolvers.

-  Post: {
-    content: (parent) => parent.content,
-    id: (parent) => parent.id,
-    published: (parent) => parent.published,
-    title: (parent) => parent.title,
-  },
 
Finally, you export the schema and resolvers so that you can use them in the next step to instantiate the server with Apollo Server.

## Step 3 — Creating the GraphQL Server
In this step, you will create the GraphQL server with Apollo Server and bind it to a port so that the server can accept connections.

First, run the following command to create the file for the server:

```java
nano src/server.js
```
 
Now add the following code to the file:

prisma-graphql/src/server.js
```java
const { ApolloServer } = require('apollo-server')
const { resolvers, typeDefs } = require('./schema')

const port = process.env.PORT || 8080

new ApolloServer({ resolvers, typeDefs }).listen({ port }, () =>
  console.log(`Server ready at: http://localhost:${port}`),
)
```
 
Here you instantiate the server and pass the schema and resolvers from the previous step.

The port the server will bind to is set from the PORT environment variable and if not set, it will default to 8080. The PORT environment variable will be automatically set by App Platform and ensure your server can accept connections once deployed.

Save and exit the file.

Your GraphQL API is ready to run. Start the server with the following command:

```java
node src/server.js
```
 
You will receive the following output:

Output
```java
Server ready at: http://localhost:8080
```

It’s considered good practice to add a start script to your package.json so that the entry point to your server is clear. Moreover, this will allow App Platform to start the server once deployed.

To do so, add the following line to the "scripts" object in package.json:

package.json
```java
{
  "name": "prisma-graphql",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "start": "node ./src/server.js"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "dependencies": {
    "apollo-server": "^2.18.2",
    "graphql": "^15.3.0"
  }
}
```
 
Save and exit the file once you’re finished.

Now you can start the server with the following command:

```java
npm start
```
 
To test the GraphQL API, open the URL from the output, which will lead you to the GraphQL Playground.

## GraphQL Playground

The GraphQL Playground is an IDE where you can test the API by sending queries and mutations.

For example, to test the feed query which only returns published posts, enter the following query to the left side of the IDE and send the query by pressing the play button:

```java
query {
  feed {
    id
    title
    content
    published
  }
}
```
 
The response will show a title of Subscribe to GraphQL Weekly with its URL and Follow DigitalOcean on Twitter with its URL.

## GraphQL query

To test the createDraft mutation, enter the following mutation:

```java
mutation {
  createDraft(title: "Deploying a GraphQL API to DigitalOcean") {
    id
    title
    content
    published
  }
}
```
 
After you submit the mutation, using the play button, you will receive Deploying a GraphQL API to DigitalOcean within the title field as part of the response.

## GraphQL mutation

Note: You can choose which fields to return from the mutation by adding or removing fields within the curly braces following createDraft. For example, if you wanted to only return the id and title you could send the following mutation:

```java
mutation {
  createDraft(title: "Deploying a GraphQL API to DigitalOcean") {
    id
    title
  }
}
```
 
You have successfully created and tested the GraphQL server. In the next step, you will create a GitHub repository for the project.

## Step 4 — Creating the GitHub Repository
In this step, you will create a GitHub repository for your project and push your changes so that the GraphQL API can be automatically deployed from GitHub to App Platform.

Begin by initializing a repository from the prisma-graphql folder:

```java
git init
```
 
Next, use the following two commands to commit the code to the repository:

```java
git add src package-lock.json package.json
git commit -m 'Initial commit'
```
 
Now that the changes have been committed to your local repository, you will create a repository in GitHub and push your changes.

Go to GitHub to create a new repository. For consistency, name the repository prisma-graphql and then click Create repository.

After the repository was created, push the changes with the following commands, which includes renaming the default local branch to main:

```java
git remote add origin git@github.com:your_github_username/prisma-graphql.git
git branch -M main
git push --set-upstream origin main
```
 
You have successfully committed and pushed the changes to GitHub. Next, you will connect the repository to App Platform and deploy the GraphQL API.

## Step 5 — Deploying to App Platform
In this step, you will connect the GitHub repository you created in the previous step to DigitalOcean and configure App Platform so that the GraphQL API can be automatically deployed when you push changes to GitHub.

First, go to the App Platform and click on the Launch Your App button.

You will see a button to link your GitHub account.

## Link Your GitHub Acccount

Click on it, and you will be redirected to GitHub.

Click Install & Authorize and you will be redirected back to DigitalOcean.

Choose the repository your_github_username/prisma-graphql and click Next.

Name your app and pick a branch to deploy from

Choose the region you want to deploy your app to and click Next.

## Configure your app

Here you can customize the configuration for the app. Ensure that the Run Command is npm start. By default, App Platform will set the HTTP port to 8080, which is the same port that you’ve configured your GraphQL server to bind to.

Click Next and you will be prompted to pick the plan.

Select Basic and then click Launch Basic App. You will be redirected to the app page, where you will see the progress of the initial deployment.

Once the build finishes, you will get a notification indicating that your app is deployed.

## App page

You can now visit your deployed GraphQL API at the URL below the app’s name. It will be under the ondigitalocean.app subdomain. If you open the URL, the GraphQL Playground will open the same way as it did in Step 3 of the tutorial.

You have successfully connected your repository to App Platform and deployed your GraphQL API. Next you will evolve your app and replace the in-memory data of the GraphQL API with a database.

## Step 6 — Setting Up Prisma with PostgreSQL
So far the GraphQL API you built used the in-memory posts array to store data. This means that if your server restarts, all changes to the data will be lost. To ensure that your data is safely persisted, you will replace the posts array with a PostgreSQL database and use Prisma to access the data.

In this step, you will install the Prisma CLI, create your initial Prisma schema, set up PostgreSQL locally with Docker, and connect Prisma to it.

The Prisma schema is the main configuration file for your Prisma setup and contains your database schema.

Begin by installing the Prisma CLI with the following command:

```java
npm install --save-dev @prisma/cli
```
 
The Prisma CLI will help with database workflows such as running database migrations and generating Prisma Client.

Next, you’ll set up your PostgreSQL database using Docker. Create a new Docker Compose file with the following command:

```java
nano docker-compose.yml
```
 
Now add the following code to the newly created file:

prisma-graphql/docker-compose.yml
```java
version: '3.8'
services:
  postgres:
    image: postgres:10.3
    restart: always
    environment:
      - POSTGRES_USER=test-user
      - POSTGRES_PASSWORD=test-password
    volumes:
      - postgres:/var/lib/postgresql/data
    ports:
      - '5432:5432'
volumes:
  postgres:
```
 
This Docker Compose configuration file is responsible for starting the official PostgreSQL Docker image on your machine. The POSTGRES_USER and POSTGRES_PASSWORD environment variables set the credentials for the superuser (a user with admin privileges). You will also use these credentials to connect Prisma to the database. Finally, you define a volume where PostgreSQL will store its data, and bind the 5432 port on your machine to the same port in the Docker container.

Save and exit the file.

With this setup in place, go ahead and launch the PostgreSQL database server with the following command:

```java
docker-compose up -d
```
 
You can verify that the database server is running with the following command:

```java
docker ps
```
 
This will output something similar to:

Output
```java
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
198f9431bf73        postgres:10.3       "docker-entrypoint.s…"   45 seconds ago      Up 11 seconds       0.0.0.0:5432->5432/tcp   prisma-graphql_postgres_1
```

With the PostgreSQL container running, you can now create your Prisma setup. Run the following command from the Prisma CLI:

```java
npx prisma init
```
 
Note that as a best practice, all invocations of the Prisma CLI should be prefixed with npx. This ensures it’s using your local installation.

After running the command, the Prisma CLI created a new folder called prisma in your project. It contains the following two files:

- schema.prisma: The main configuration file for your Prisma project (in which you will include your data model).
- .env: A dotenv file to define your database connection URL.
To make sure Prisma knows about the location of your database, open the .env file:

```java
nano prisma/.env
```
 
Adjust the DATABASE_URL environment variable to look as follows:

prisma-graphql/prisma/.env
```java
DATABASE_URL="postgresql://test-user:test-password@localhost:5432/my-blog?schema=public"
```
 
Note that you’re using the database credentials test-user and test-password, which are specified in the Docker Compose file. To learn more about the format of the connection URL, visit the Prisma docs.

You have successfully started PostgreSQL and configured Prisma using the Prisma schema. In the next step, you will define your data model for the blog and use Prisma Migrate to create the database schema.

## Step 7 — Defining the Data Model with Prisma Migrate
Now you will define your data model in the Prisma schema file you’ve just created. This data model will then be mapped to the database with Prisma Migrate, which will generate and send the SQL statements for creating the tables that correspond to your data model.

Since you’re building a blog, the main entities of the application will be users and posts. In this step, you will define a Post model with a similar structure to the Post type in the GraphQL schema. In a later step, you will evolve the app and add a User model.

Note: The GraphQL API can be seen as an abstraction layer for your database. When building a GraphQL API, it’s common for the GraphQL schema to closely resemble your database schema. However, as an abstraction, the two schemas won’t necessarily have the same structure, thereby allowing you to control which data you want to expose over the API. This is because some data might be considered sensitive or irrelevant for the API layer.

Prisma uses its own data modeling language to define the shape of your application data.

Open your schema.prisma file from the project’s folder where package.json is located:

```java
nano prisma/schema.prisma
```
 
Note: You can verify from the terminal in which folder you are with the pwd command, which will output the current working directory. Additionally, listing the files with the ls command will help you navigate your file system.

Add the following model definitions to it:

prisma-graphql/prisma/schema.prisma
```java
...
model Post {
  id        Int     @default(autoincrement()) @id
  title     String
  content   String?
  published Boolean @default(false)
}
```
 
You are defining a model called Post with a number of fields. The model will be mapped to a database table; the fields represent the individual columns.

The id fields have the following field attributes:

- @default(autoincrement()): This sets an auto-incrementing default value for the column.
- @id: This sets the column as the primary key for the table.

Save and exit the file once you’re done.

With the model in place, you can now create the corresponding table in the database using Prisma Migrate. This can be done with the migrate dev command that creates the migration files and runs them.

Open up your terminal again and run the following command:

```java
npx prisma migrate dev --preview-feature --name "init" --skip-generate
```
 
This will output something similar to:

Output

```java
PostgreSQL database my-blog created at localhost:5432

Prisma Migrate created and applied the following migration(s) from new schema changes:

migrations/
  └─ 20201201110111_init/
    └─ migration.sql

Everything is now in sync.
```

This command creates a new migration on your file system and runs it against the database to create the database schema. Here’s a quick overview of the options that are provided to the command:

- --preview-feature: Required because Prisma Migrate is currently in a preview state.
- --name "init": Specifies the name of the migration (will be used to name the migration folder that’s created on your file system).
- --skip-generate: Skips generating Prisma Client (this will be done in the next step).
Your prisma/migrations directory is now populated with the SQL migration file. This approach allows you to track changes to the database schema and create the same database schema in production.

Note: If you’ve already used Prisma Migrate with the my-blog database and there is an inconsistency between the migrations in the prisma/migration folder and the database schema you will be prompted to reset the database with the following output:

Output
```java
? We need to reset the PostgreSQL database "my-blog" at "localhost:5432". All data will be lost.
Do you want to continue? › (y/N)
```

You can resolve this by entering y which will reset the database. Beware that this will cause all data in the database to be lost.

You’ve now created your database schema. In the next step, you will install Prisma Client and use it in your GraphQL resolvers.

## Step 8 — Using Prisma Client in the GraphQL Resolvers
Prisma Client is an auto-generated and type-safe Object Relational Mapper (ORM) that you can use to programmatically read and write data in a database from a Node.js application. In this step, you’ll install Prisma Client in your project.

Open up your terminal again and install the Prisma Client npm package:

```java
npm install @prisma/client
```
 
Note: Prisma Client gives you rich auto-completion by generating code based on your Prisma schema to the node_modules folder. To generate the code you use the npx prisma generate command. This is typically done after you create and run a new migration. On the first install, however, this is not necessary as it will automatically be generated for you in a postinstall hook.

With the database and GraphQL schema created, and Prisma Client installed, you will now use Prisma Client in the GraphQL resolvers to read and write data in the database. You’ll do this by replacing the posts array, which you’ve used so far to hold your data.

Begin by creating the following file:

```java
nano src/db.js
```
 
Add the following:

prisma-graphql/src/db.js
```java
const { PrismaClient } = require('@prisma/client')

module.exports = {
  prisma: new PrismaClient(),
}
```
 
This imports Prisma Client, creates an instance of it, and exports the instance that you’ll use in your resolvers.

Now save and close the src/db.js file.

Next, you will import the prisma instance into src/schema.js. To do so, open src/schema.js:

```java
nano src/schema.js
```
 
Then import prisma from ./db at the top of the file:

prisma-graphql/src/schema.js
```java
const { prisma } = require('./db')
...
```
 
Then delete the posts array:

prisma-graphql/src/schema.js
```java
-const posts = [
-  {
-    id: 1,
-    title: 'Subscribe to GraphQL Weekly for community news ',
-    content: 'https://graphqlweekly.com/',
-    published: true,
-  },
-  {
-    id: 2,
-    title: 'Follow DigitalOcean on Twitter',
-    content: 'https://twitter.com/digitalocean',
-    published: true,
-  },
-  {
-    id: 3,
-    title: 'What is GraphQL?',
-    content: 'GraphQL is a query language for APIs',
-    published: false,
-  },
-]
```
 
Now you will update the Query resolvers to fetch published posts from the database. Update the resolvers.Query object with the following resolvers:

prisma-graphql/src/schema.js
```java
...
const resolvers = {
  Query: {
    feed: (parent, args) => {
      return prisma.post.findMany({
        where: { published: true },
      })
    },
    post: (parent, args) => {
      return prisma.post.findOne({
        where: { id: Number(args.id) },
      })
    },
  },
```
 
Here, you’re using two Prisma Client queries:

- findMany: Fetches posts whose publish field is false.
- findOne: Fetches a single post whose id field equals the id GraphQL argument.
- Note, that as per the GraphQL specification, the ID type is serialized the same way as a String. Therefore you convert to a Number because the id in the Prisma schema is an int.

Next, you will update the Mutation resolver to save and update posts in the database. Update the resolvers.Mutation Object with the following resolvers:

prisma-graphql/src/schema.js
```java
const resolvers = {
  ...
  Mutation: {
    createDraft: (parent, args) => {
      return prisma.post.create({
        data: {
          title: args.title,
          content: args.content,
        },
      })
    },
    publish: (parent, args) => {
      return prisma.post.update({
        where: {
          id: Number(args.id),
        },
        data: {
          published: true,
        },
      })
    },
  },
}
```
 
You’re using two Prisma Client queries:

- create: Create a Post record.
- update: Update the published field of the Post record whose id matches the one in the query argument.
Your schema.js should now look as follows:

prisma-graphql/src/schema.js
```java
const { gql } = require('apollo-server')
const { prisma } = require('./db')

const typeDefs = gql`
  type Post {
    content: String
    id: ID!
    published: Boolean!
    title: String!
  }

  type Query {
    feed: [Post!]!
    post(id: ID!): Post
  }

  type Mutation {
    createDraft(content: String, title: String!): Post!
    publish(id: ID!): Post
  }
`

const resolvers = {
  Query: {
    feed: (parent, args) => {
      return prisma.post.findMany({
        where: { published: true },
      })
    },
    post: (parent, args) => {
      return prisma.post.findOne({
        where: { id: Number(args.id) },
      })
    },
  },
  Mutation: {
    createDraft: (parent, args) => {
      return prisma.post.create({
        data: {
          title: args.title,
          content: args.content,
        },
      })
    },
    publish: (parent, args) => {
      return prisma.post.update({
        where: {
          id: Number(args.id),
        },
        data: {
          published: true,
        },
      })
    },
  },
}

module.exports = {
  resolvers,
  typeDefs,
}
```
 
Save and close the file.

Now that you’ve updated the resolvers to use Prisma Client, start the server to test the flow of data between the GraphQL API and the database with the following command:

```java
npm start
```
 
Once again, you will receive the following output:

Output
```java
Server ready at: http://localhost:8080
```

Open the GraphQL playground at the address from the output and test the GraphQL API using the same queries from Step 3.

Now you will commit your changes so that the changes can be deployed to App Platform.

To avoid committing the node_modules folder and the prisma/.env file, begin by creating a .gitignore file:

```java
nano .gitignore
```
 
Add the following to the file:

prisma-graphql/.gitignore
```java
node_modules
prisma/.env
```
  
Save and exit the file.

Then run the following two commands to commit the changes:

```java
git add .
git commit -m 'Add Prisma'
```
 
Next you’ll add a PostgreSQL database to your app in App Platform.

## Step 9 — Creating and Migrating the PostgreSQL Database in App Platform
In this step, you will add a PostgreSQL database to your app in App Platform. Then you will use Prisma Migrate to run the migration against it so that the deployed database schema matches your local database.

First, go to the App Platform console and choose the prisma-graphql project you created in Step 5.

Next, go to the Components tab.

## Component Tab

Click on + Create Component and select Database, which will lead you to a page to configure your database.

## Configure your database

Choose Dev Database and click Create and Attach.

You will be redirected back to the Components page, where there will be a progress bar for creating the database.

## Creating database

After the database has been created, you will run the database migration against the production database on DigitalOcean from your local machine. To run the migration, you will need the connection string of the hosted database. To get it, click on the db icon in the components tab.

## Database information

Under Connection Details select Connection String in the dropdown and copy the database URL, which will have the following structure:

```java
postgresql://db:some_password@unique_identifier.db.ondigitalocean.com:25060/db?sslmode=require
```

Then, run the following command in your terminal and ensure that you set DATABASE_URL to the same URL you copied:

```java
DATABASE_URL="postgresql://db:some_password@unique_identifier.db.ondigitalocean.com:25060/db?sslmode=require" npx prisma migrate deploy --preview-feature
```
 
This will run the migrations against the live database with Prisma Migrate.

If the migration succeeds you will receive the following:

Output
```java
PostgreSQL database db created at unique_identifier.db.ondigitalocean.com:25060

Prisma Migrate applied the following migration(s):

migrations/
  └─ 20201201110111_init/
    └─ migration.sql
You have successfully migrated the production database on DigitalOcean, which now matches the Prisma schema.
```

Now you can deploy your app by pushing your Git changes with the following command:

```java
git push
```
 
Note: App Platform will make the DATABASE_URL environment variable available to your application at run-time. Prisma Client will use that environment variable using the env("DATABASE_URL") in the datasource block of your Prisma schema.

This will automatically trigger a build. If you open App Platform console, you will have a Deploying progress bar.

## Deploying

Once the deployment succeeds, you will receive a Deployed successfully message.

You’ve now backed up your deployed GraphQL API with a database. Open the Live App, which will lead you to the GraphQL Playground. Test the GraphQL API using the same queries from Step 3.

In the final step you will evolve the GraphQL API by adding the User model.

## Step 10 — Adding the User Model
Your GraphQL API for blogging has a single entity named Post. In this step, you’ll evolve the API by defining a new model in the Prisma schema and adapting the GraphQL schema to make use of the new model. You will introduce a User model with a one-to-many relation to the Post model. This will allow you to represent the author of posts and associate multiple posts to each user. Then you will evolve the GraphQL schema to allow creating users and associating posts with users through the API.

First, open the Prisma schema and add the following:

prisma-graphql/prisma/schema.prisma
```java
...
model Post {
  id        Int     @id @default(autoincrement())
  title     String
  content   String?
  published Boolean @default(false)
  author    User?   @relation(fields: [authorId], references: [id])
  authorId  Int?
}

model User {
  id    Int    @id @default(autoincrement())
  email String @unique
  name  String
  posts Post[]
}
```
 
You’ve added the following to the Prisma schema:

- The User model to represent users.
- Two relation fields: author and posts. Relation fields define connections between models at the Prisma level and do not exist in the database. These fields are used to generate the Prisma Client and to access relations with Prisma Client.
- The authorId field, which is referenced by the @relation attribute. Prisma will create a foreign key in the database to connect Post and User.
- Note that the author field in the Post model is optional. That means you’ll be able to create posts unassociated with a user.

Save and exit the file once you’re done.

Next, create and apply the migration locally with the following command:

```java
npx prisma migrate dev --preview-feature --name "add-user"
```
 
If the migration succeeds you will receive the following:

Output
```java
Prisma Migrate created and applied the following migration(s) from new schema changes:

migrations/
  └─ 20201201123056_add_user/
    └─ migration.sql

✔ Generated Prisma Client to ./node_modules/@prisma/client in 53ms
```

The command also generates Prisma Client so that you can make use of the new table and fields.

Now you will run the migration against the production database on App Platform so that the database schema is the same as your local database. Run the following command in your terminal and set DATABASE_URL to the connection URL from App Platform:

```java
DATABASE_URL="postgresql://db:some_password@unique_identifier.db.ondigitalocean.com:25060/db?sslmode=require" npx prisma migrate deploy --preview-feature
```
 
You will receive the following:

Output
```java
Prisma Migrate applied the following migration(s):

migrations/
  └─ 20201201123056_add_user/
    └─ migration.sql
```
You will now update the GraphQL schema and resolvers to make use of the updated database schema.

Open the src/schema.js file and add update typeDefs as follows:

prisma-graphql/src/schema.js
```java
...
const typeDefs = gql`
  type User {
    email: String!
    id: ID!
    name: String
    posts: [Post!]!
  }

  type Post {
    content: String
    id: ID!
    published: Boolean!
    title: String!
    author: User
  }

  type Query {
    feed: [Post!]!
    post(id: ID!): Post
  }

  type Mutation {
    createUser(data: UserCreateInput!): User!
    createDraft(authorEmail: String, content: String, title: String!): Post!
    publish(id: ID!): Post
  }

  input UserCreateInput {
    email: String!
    name: String
    posts: [PostCreateWithoutAuthorInput!]
  }

  input PostCreateWithoutAuthorInput {
    content: String
    published: Boolean
    title: String!
  }
`
...
```
 
In this updated code, you’re adding the following changes to the GraphQL schema:

- The User type, which returns an array of Post.
- The author field to the Post type.
- The createUser mutation, which expects the UserCreateInput as its input type.
- The PostCreateWithoutAuthorInput input type used in the UserCreateInput input for creating posts as part of the createUser mutation.
- The authorEmail optional argument to the createDraft mutation.
With the schema updated, you will now update the resolvers to match the schema.

Update the resolvers object as follows:

prisma-graphql/src/schema.js
```java
...
const resolvers = {
  Query: {
    feed: (parent, args) => {
      return prisma.post.findMany({
        where: { published: true },
      })
    },
    post: (parent, args) => {
      return prisma.post.findOne({
        where: { id: Number(args.id) },
      })
    },
  },
  Mutation: {
    createDraft: (parent, args) => {
      return prisma.post.create({
        data: {
          title: args.title,
          content: args.content,
          published: false,
          author: args.authorEmail && {
            connect: { email: args.authorEmail },
          },
        },
      })
    },
    publish: (parent, args) => {
      return prisma.post.update({
        where: { id: Number(args.id) },
        data: {
          published: true,
        },
      })
    },
    createUser: (parent, args) => {
      return prisma.user.create({
        data: {
          email: args.data.email,
          name: args.data.name,
          posts: {
            create: args.data.posts,
          },
        },
      })
    },
  },
  User: {
    posts: (parent, args) => {
      return prisma.user
        .findOne({
          where: { id: parent.id },
        })
        .posts()
    },
  },
  Post: {
    author: (parent, args) => {
      return prisma.post
        .findOne({
          where: { id: parent.id },
        })
        .author()
    },
  },
}
```
 
Let’s break down the changes to the resolvers:

- The createDraft mutation resolver now uses the authorEmail argument (if passed) to create a relation between the created draft and an existing user.
- The new createUser mutation resolver creates a user and related posts using nested writes.
- The User.posts and Post.author resolvers define how to resolve the posts and author fields when the User or Post are queried. These use Prisma’s Fluent API to fetch the relations.
Save and exit the file.

Start the server to test the GraphQL API:

```java
npm start
```
 
Begin by testing the createUser resolver with the following GraphQL mutation:

```java
mutation {
  createUser(data: { email: "natalia@prisma.io", name: "Natalia" }) {
    email
    id
  }
}
 
This will create a user.

Next, test the createDraft resolver with the following mutation:

mutation {
  createDraft(
    authorEmail: "natalia@prisma.io"
    title: "Deploying a GraphQL API to App Platform"
  ) {
    id
    title
    content
    published
    author {
      id
      name
    }
  }
}
```
 
Notice that you can fetch the author whenever the return value of a query is Post. In this example, the Post.author resolver will be called.

Finally, commit your changes and push to deploy the API:

```java
git add .
git commit -m "add user model"
git push
```
 
You have successfully evolved your database schema with Prisma Migrate and exposed the new model in your GraphQL API.

## Conclusion
In this article, you built a GraphQL API with Prisma and GraphQL, and deployed it to DigitalOcean’s App Platform. You defined a GraphQL schema and resolvers with Apollo Server. You then used Prisma Client in your GraphQL resolvers to persist and query data in the PostgreSQL database.

As a next step, you can further extend the GraphQL API with a query to fetch individual users and a mutation to connect an existing draft to a user.

If you’re interested in exploring the data in the database, check out Prisma Studio. Be sure to visit the Prisma documentation to learn about different aspects of Prisma and explore some ready-to-run example projects in the prisma-examples repository.

You can find the code for this project in the DigitalOcean Community repository. 
https://github.com/do-community/prisma-graphql
