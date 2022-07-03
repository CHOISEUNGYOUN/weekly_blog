# Handling GraphQL errors like a champ with unions and interfaces

Without a doubt, one of the best features of GraphQL is its awesome type system.

Together with tools like the GraphQL Code Generator and typed Javascript subsets like TypeScript or Flow, you can generate fully typed data fetching code within seconds.

I cannot think back to the time where I had to design and build API’s without the GraphQL ecosystem.

When I started using GraphQL, I had some issues with changing the mindset I’d developed by thinking in REST.

One thing I’ve been particularly displeased about is error handling. In traditional HTTP, you have different Status Codes that represent different types of errors (or successes).

When GraphQL was gaining popularity, I remember a meme made of some terminal that showed an Apollo server logging an error object with status code 200 and the caption `ok`. I was wondering why GraphQL breaks these widely-used standards.

Today, I know that GraphQL gives us the power to handle errors in a better and more explicit way.

## Handling errors in GraphQL
Before we take a look at how I design my APIs today, I wanna showcase the evolution of how I was handling errors until recently.

I’ll use `react-apollo` and `apollo-server` code examples throughout this article. However, the concepts should be applicable to any other client and server framework.

Let’s start with a look at the following JSON object:

```json
{
  "errors": [
    {
      "message": "Name for character with ID 1002 could not be fetched.",
      "locations": [ { "line": 6, "column": 7 } ],
      "path": [ "hero", "heroFriends", 1, "name" ]
    }
  ],
  "data": {
    "hero": {
      "name": "R2-D2",
      "heroFriends": [
        {
          "id": "1000",
          "name": "Luke Skywalker"
        },
        {
          "id": "1002",
          "name": null
        },
        {
          "id": "1003",
          "name": "Leia Organa"
        }
      ]
    }
  }
}
```
Does this seem familiar?

This exact code is copied from the [GraphQL Spec Error Section](https://spec.graphql.org/draft/#example-90475). If you’ve already integrated a GraphQL API into your application, you may be familiar with this response format.

By design, GraphQL has the capabilities to declare fields nullable. Despite this data being optional, it also allows us to send partial results if a resolver throws an error.

This is one thing that differentiates GraphQL from strict REST.

If a resolver throws an error — in this case, the name resolver for the hero with the id 1002 — a new array with the key errors is appended to the response JSON object.

The array contains an error object with the original message of the error, a path, and a query location.

The code for the resolver would look similar to this:

```js
const resolvers = {
  Hero: {
    name: (parent, args, context) => {
      throw new Error(
        "Name for character with ID 1002 could not be fetched."
      );
    },
  },
};
```
I once thought this was pretty cool.

Then I realized I needed more detailed info — something like a status code or an error code. How would I distinguish a “user does not exist” error from a “user has blocked you” error?

The community learned, and the concept of extensions was added to the GraphQL spec.

Extensions are nothing more than an additional object that can be added to your error object (or response object).

```json
{
  "errors": [
    {
      "message": "Name for character with ID 1002 could not be fetched.",
      "locations": [ { "line": 6, "column": 7 } ],
      "path": [ "hero", "heroFriends", 1, "name" ],
      "extensions": {
        "code": "CAN_NOT_FETCH_BY_ID",
        "timestamp": "Fri Feb 9 14:33:09 UTC 2018"
      }
    }
  ]
}
```

With `extensions`, we can add a `code` property to our error object, which can then be used by the client (e.g. a `switch` or `if` statement).

This is way more convenient than parsing the error message for interpreting the error.

Frameworks like the Apollo Server provide Error classes that can be initialized with an error message and a code:

```js
import {
  ApolloError,
}  from "apollo-server";

const resolvers = {
  Hero: {
    name: (parent, args, context) => {
      throw new ApolloError(
        "Name for character with ID 1002 could not be fetched.",
        "CAN_NOT_FETCH_BY_ID",
      );
    },
  },
};
```

Of course, I also started quickly adopting this style of error handling, but I soon realized that there are some drawbacks that reduce my productivity:

## The errors are not collocated to where they occur

Of course, you have a path array that describes where an error occurs (e.g. `[ hero, heroFriends, 1, name ]`). You can build some custom function in your client that maps an error to your query path.

I personally believe that every error should be handled in the UI of the application.

Having the error located somewhere different by default doesn’t really encourage developers to gracefully handle errors.

Furthermore, frameworks like relay modern encourage you to only inject fragments into your components.

For proper error handling, you need to apply custom logic for injecting the correct error into the correct component.

Sounds like extra work that I personally would want to avoid.

## Using the errors robs us of one of the main benefits of GraphQL: type safety
As mentioned earlier, one of the main benefits of a GraphQL API is type safety.

A schema is by default introspectable and exposes a complete register of all the available types and fields.

Unfortunately, the error codes don’t follow any schema (at least not according to the GraphQL spec).

No type error will be thrown if you mistype the error message or extension code inside your resolvers.

The GraphQL engine does not care about the structure of the message.

Furthermore, the error code is just an optional extension. I am not currently aware of any tool that generates type-safe error codes, nor can you see an overview of all available error codes that a field (or resolver) could throw.

## When using the errors array, we’re back in good old type guessing land.
Backend and frontend developers now have one more pain to deal with (one that they actually tried to avoid by switching over to GraphQL in the first place.)

Don’t misunderstand me — even if you have a fully-typed GraphQL API, there should still be some documentation.

The API browser generated by tools like GraphiQL or GraphQL Playground should make it easier to discover and understand what a GraphQL API provides, but it shouldn’t replace a documentation with usage examples.

## We can do it better with the existing GraphQL primitives
Recently, there has been a lot of buzz around using union types for handling errors. A union type represents a list of objects that a field can return.

```graphql
type User {
  id: ID!
  login: String!
}

type UserNotFoundError {
  message: String!
}

union UserResult = User | UserNotFoundError

type Query {
  user(id: ID!): UserResult!
}
```

In the following schema, the field `user` can either return a `User` or `UserNotFoundError`. Instead of throwing an error inside our resolver, we simply return a different type.

The query that you would send to your server would look like this:

```graphql
query user($id: ID!) {
  user(id: $id) {
    ... on UserNotFoundError {
      message
    }
    ... on User {
      id
      login
    }
  }
}
```
Accordingly, the `apollo-server` resolver could look similar to the following:

```js
const resolvers = {
  Query: {
    user: async (parent, args, context) => {
      const userRecord = await context.db.findUserById(args.id);
      if (userRecord) {
        return {
          __typename: "User",
          ...userRecord,
        };
      }
      return {
        __typename: "UserNotFound",
        message: `The user with the id ${args.id} does not exist.`,
      };
    },
  },
};
```

When using unions, you will have to return a `__typename` so `apollo-server` knows which type the result has and which resolver map must be used for resolving further field values of the resolved type.

This allows us to model errors like normal GraphQL types. This way, we regain the power of type safety: instead of working with a message and an error code, we can have more complex types.

Below is an example of a login mutation that returns the `UserRegisterInvalidInputError` error type.

Despite having a generic error message, the type also provides fields for the single input fields.

```graphql
type User {
  id: ID!
  login: String!
}

type UserRegisterResultSuccess {
  user: User!
}

type UserRegisterInvalidInputError {
  message: String!
  loginErrorMessage: String
  emailErrorMessage: String
  passwordErrorMessage: String
}

input UserRegisterInput {
  login: String!
  email: String!
  password: String!
}

union UserRegisterResult = UserRegisterResultSuccess | UserRegisterInvalidInputError

type Mutation {
  userRegister(input: UserRegisterInput!): UserRegisterResult!
}
```

You could even go further and add fields that return new, more complex `object` types.

A client implementation could look like similar to this:

```js
import React, { useState } from "react";
import { useUserRegisterMutation } from "./generated-types"
import idx from "idx";
import { useFormState } from 'react-use-form-state';

const RegistrationForm: React.FC<{}> = () => {
  const [userRegister, { loading, data }] = useUserRegisterMutation();
  const loginState = useFormState("login");
  const emailState = useFormState("email");
  const passwordState = useFormState("password");

  useEffect(() => {
    if (idx(data, d => d.userRegister.__typename) === "UserRegisterResultSuccess") {
      alert("registration success!");
    }
  }, [data]);

  return (
    <form
      onSubmit={(ev) => {
        ev.preventDefault();
        userRegister();
      }}
    >
      <InputField
        {...loginState}
        error={idx(data, d => d.userRegister.loginErrorMessage)}
      />
      <InputField
        {...emailState}
        error={idx(data, d => d.userRegister.emailErrorMessage)}
      />
      <InputField
        {...passwordState}
        error={idx(data, d => d.userRegister.passwordErrorMessage)}
      />
      <SubmitButton />
      {idx(data, d => d.userRegister.message) || null}
      {loading ? <LoadingSpinner /> : null}
    </form>
  )
}
```

## GraphQL gives you the power to shape your data tree according to your UI
That’s why you should also shape your error types according to the UI.

In case you have different types of errors, you can create a type for each of them and add them to your union list:

```graphql
type User {
  id: ID!
  login: String!
}

type UserRegisterResultSuccess {
  user: User!
}

type UserRegisterInvalidInputError {
  message: String!
  loginErrorMessage: String
  emailErrorMessage: String
  passwordErrorMessage: String
}

type CountryBlockedError {
  message: String!
}

type UserRegisterInput {
  login: String!
  email: String!
  password: String!
}

union UserRegisterResult =
  UserRegisterResultSuccess
  | UserRegisterInvalidInputError
  | CountryBlockedError

type Mutation {
  userRegister(input: UserRegisterInput!): UserRegisterResult!
}
```

This allows each error type to have their unique properties.

Let’s jump over the the frontend part of this requirement:

> You have a new requirement for your API: people from country X should not be allowed to register anymore, due to some weird sanctions of the country your company operates from.

Seems pretty straightforward, just add some new types on the backend, right?

Unfortunately, no. The frontend developer will now also have to update his query because a new type of error, that is not covered by any selection set is now being returned.

This means that the following query:

```graphql
mutation userRegister($input: UserRegisterInput!) {
  userRegister(input: $input) {
    __typename
    ... on UserRegisterResultSuccess {
      user {
        id
        login
      }
    }
    ... on UserRegisterInvalidInputError {
      message
      loginErrorMessage
      emailErrorMessage
      passwordErrorMessage
    }
  }
}
```
Needs to be updated to this:

```graphql
mutation userRegister($input: UserRegisterInput!) {
  userRegister(input: $input) {
    __typename
    ... on UserRegisterResultSuccess {
      user {
        id
        login
      }
    }
    ... on UserRegisterInvalidInputError {
      message
      loginErrorMessage
      emailErrorMessage
      passwordErrorMessage
    }
    ... on CountryBlockedError {
      message
    }
  }
}
```
Otherwise, the client will not receive an error message for the `CountryBlockedError` that can be displayed.

Forcing the developer of the client application to adjust their GraphQL documents every time we add some new error type does not seem like a clever solution.

Let’s take a closer look at our error objects:

```graphql
type UserRegisterInvalidInputError {
  message: String!
  loginErrorMessage: String
  emailErrorMessage: String
  passwordErrorMessage: String
}

type CountryBlockedError {
  message: String!
}
```
They both have one common property: `message`

Furthermore, we might assume that every error that will be potentially added to a union in the future will also have a message property.

Fortunately, GraphQL provides us with `interfaces`, that allow us to describe such an abstraction.

```graphql
interface Error {
  message: String!
}
```

An interface describes fields that can be implemented/shared by different types:

```graphql
interface Node {
  id: ID!
}

type User implements Node {
  id: ID!
  login: String!
}

type Post implements Node {
  id: ID!
  title: String!
  body: String!
}

type Query {
  entity(id: ID!): Node
}
```

For queries, the power of interfaces lies in being able to declare a data selection trough an interface instead of a type.

That means our previous schema can be transformed into the following:

```graphql
type User {
  id: ID!
  login: String!
}

interface Error {
  message: String!
}

type UserRegisterResultSuccess {
  user: User!
}

type UserRegisterInvalidInputError implements Error {
  message: String!
  loginErrorMessage: String
  emailErrorMessage: String
  passwordErrorMessage: String
}

type CountryBlockedError implements Error {
  message: String!
}

type UserRegisterInput {
  login: String!
  email: String!
  password: String!
}

union UserRegisterResult =
  UserRegisterResultSuccess
  | UserRegisterInvalidInputError
  | CountryBlockedError

type Mutation {
  userRegister(input: UserRegisterInput!): UserRegisterResult!
}
```

Both error types now implement the Error interface.

We can now adjust our query to the following:

```graphql
mutation userRegister($input: UserRegisterInput!) {
  userRegister(input: $input) {
    __typename
    ... on UserRegisterResultSuccess {
      user {
        id
        login
      }
    }
    ... on Error {
      message
    }
    ... on UserRegisterInvalidInputError {
      loginErrorMessage
      emailErrorMessage
      passwordErrorMessage
    }
  }
}
```

No need to even declare the `CountryBlockedError` selection set anymore. It is automatically covered by the Error selection set.

Furthermore, if any new type that implements the `Error` interface is added to the `UserRegisterResult` union, the error message will be automatically included in the result.

Of course, you will still have to add some logic on the client for handling your error state, but instead of explicitly handling every single error you can switch between the ones that need some more work, like `UserRegisterInvalidInputError`, and all these other errors that only show some sort of dialog, like `CountryBlockedError`.

E.g. if you follow the convention of ending all your error type with the word `Error` , you can build an abstraction that will handle multiple error types.

```js
import React, { useState } from "react";
import { useUserRegisterMutation } from "./generated-types"
import idx from "idx";
import { useAlert } from "./alert";

const RegistrationForm: React.FC<{}> = () => {
  const [userRegister, { loading, data }] = useUserRegisterMutation();
  const loginState = useFormState("login");
  const emailState = useFormState("email");
  const passwordState = useFormState("password");
  const showAlert = useAlert();

  useEffect(() => {
    const typename = idx(data, d => d.userRegister.__typename)
    if (typename === "UserRegisterResultSuccess") {
      alert("registration success!");
    } else if (typename.endsWith("Error")) {
      showAlert(data.userRegister.message);
    }
  }, [data]);

  return (
    <form
      onSubmit={(ev) => {
        ev.preventDefault();
        userRegister();
      }}
    >
      <InputField
        {...loginState}
        error={idx(data, d => d.userRegister.loginErrorMessage)}
      />
      <InputField
        {...emailState}
        error={idx(data, d => d.userRegister.emailErrorMessage)}
      />
      <InputField
        {...passwordState}
        error={idx(data, d => d.userRegister.passwordErrorMessage)}
      />
      <SubmitButton />
      {loading ? <LoadingSpinner /> : null}
    </form>
  )
}
```

At a later point in time when your team decides that a new error should be handled different from the others, you can adjust the code by adding a new else/if statement in useEffect.