# 유니온(union)과 인터페이스(interface) 를 사용하여 GraphQL 에러 핸들링하기

의심할 여지 없이 GraphQL의 최고의 기능은 타입 시스템이다.

GraphQL 코드 제네레이터와 타입스크립트나 플로우(Flow)와 같은 자바스크립트 서브셋을 함께 활용하면 타입이 지정된 데이터를 순식간에 가져올 수 있다.

GraphQL을 사용한 이후로 GraphQL 없이 API를 개발하던 시절로 돌아갈 수 없을 지경이다. 그럼에도 불구하고 GraphQL을 처음 사용했을때 몇가지 문제점 때문에 다시 REST로 개발하고 싶은 순간들이 있었다. 그 중 하나는 바로 에러핸들링이 무척 불편하다는 점이다. 전통적인 HTTP 통신 방식에서는 각기 다른 상태 코드와 에러 메세지를 전달해줬다(또는 성공 메세지라던지).

GraphQL이 한창 인기를 얻고 있을때 아폴로 서버 터미널에서 에러 객체를 로깅할때 상태코드로 200과 함께 `ok` 라고 내려주던 밈(meme)이 유행했던 적이 있었다. 왜 GraphQL이 광범위하게 쓰이고 있던 규격을 깨고 있는지 궁금했었다. 좀 더 조사해본 뒤 GraphQL이 어떻게 에러를 좀 더 편하고 직관적인 방법으로 다룰 수 있는지 알게 되었다.

## GraphQL에서 에러 핸들링 하기

API 디자인을 어떻게 했는지 살펴보기 전에 에러 핸들링을 여기까지 하게 된 과정에 대해서 하나씩 살펴보고자 한다. 예시코드는 `react-apollo` 와 `apollo-server` 를 사용하여 구성하고자 한다. 이 두 툴을 사용하지 않더라도 GraphQL을 사용하는 어떤 클라이언트 및 서버 프레임워크에 적용 가능하니 참고하자.

우선 아래의 JSON 객체를 살펴보도록 하자.

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
어디서 한번 본것 같지 않은가? 위 코드는 [GraphQL Spec Error Section](https://spec.graphql.org/draft/#example-90475)에서 가져왔다. 이미 GraphQL API를 어플리케이션에 적용했다면 이러한 응답 포멧에 익숙할 것이다.

GraphQL은 설계상 필드 값을 nullable 처리를 할 수 있다. 해당 데이터가 선택값으로 지정되었더라도 리졸버에서 에러 메세지를 반환한다면 부분적으로 결과값을 전달 할 수 있다. 이는 엄격한 REST 구조와 다른 요소중에 하나이다.

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


Original Source:
[Handling GraphQL errors like a champ with unions and interfaces](https://blog.logrocket.com/handling-graphql-errors-like-a-champ-with-unions-and-interfaces)