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

GraphQL은 설계상 필드 값을 nullable 처리를 할 수 있다. 해당 데이터가 선택값으로 지정되었더라도 리졸버에서 에러 메세지를 반환한다면 부분적으로 결과값을 전달 할 수 있다. 이는 엄격한 REST 구조와 다른 요소중에 하나이다. `hero` 리졸버에서 에러를 반환하는 경우 (예시에서는 1002번 id) JSON 응답 객체에 key errors 배열을 추가하여 반환하게 된다. 이 배열은 어떤 에러인지에 대한 메세지를 담은 에러 객체와 경로, 쿼리 위치를 알려준다. 해당 리졸버 코드는 다음과 같다.

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
처음 이 구조를 보았을때 좋다고 생각했었다. 하지만 이내 좀 더 상태 코드와 같이 자세한 정보가 필요하다는 것을 알게 되었다. 예를 들자면 이 에러를 보고 사용자가 존재하지 않는지(REST 에서는 404) 사용자가 차단을 했는지(REST 에서는 406으로 표현 가능) 구분이 불가능한 것이다. 그리하여 GraphQL에서는 아래와 같이 익스텐션을 추가할 수 있게 되었다. 익스텐션은 말 그대로 에러 객체(또는 응답 객체)에 객체를 추가하는 것을 의미한다.

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

익스텐션을 활용하면 에러 객체에 코드를 적절히 추가할 수 있어 이를 클라이언트에서 활용할 수 있다. 이는 에러메세지를 직접 구문분석 하는 것 보다 좀 더 간편한 방법이다. Apollo 서버와 같은 프레임워크들은 에러 클래스를 선언하여 에러메세지를 돌려 주기도 한다.

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

필자 또한 위 방식과 같은 에러 핸들링 스타일을 빠르게 적용했지만 이 방법이 생산성 측면에서 장점보다는 단점이 많다는 점을 깨달았다.

## 에러는 발생이 예상되는 위치에 발생하지 않는다.

물론 어디에 이 에러가 발생하는지 알 순 있다. 클라이언트에서 에러 메세지를 찾는 커스텀 함수를 만들어서 관리를 할 수도 있다. 필자는 개인적으로 모든 에러는 어플리케이션의 UI 단에서 다뤄져야 한다고 생각한다. 에러 자체가 기본적으로 발생하는 위치 이외의 곳에서 발생하게 되는것은 개발자의 에러를 유연하게 처리하고자 하는 의지를 떨어뜨리기 마련이다. 게다가 `Relay`와 같은 프레임워크들은 `Fragment`들을 컴포넌트에 주입하도록 장려하고 있다. 에러 핸들링을 적절하게 하기 위해선 정확한 에러를 각 컴포넌트에 주입시켜주는 커스텀 로직을 구현해야 한다. 이는 필자가 피하고자 했던 추가 작업이지만 어쩔수 없게 되었다.

## GraphQL의 에러 익스텐션을 활용하는 것은 GraphQL의 타입 안정성을 해치는 작업이다.
앞서 말했듯이 GraphQL API의 가장 큰 장점 중 하나는 타입 안정성이다. 스키마는 기본적으로 자기 내부 검사 시스템을 통해 검증 될 수 있으며 모든 타입들과 필드들이 외부에 노출되어 있다. 하지만 에러 코드의 경우 스키마 내 어디에도 존재하지 않는다. (적어도 GraphQL 사양 내에서는 말이다.) 에러 메세지를 잘못 입력하거나 리졸버 내 익스텐션 코드를 잘못 입력 하더라도 아무런 타입 에러를 일으키지 않는다. GraphQL 엔진은 메세지 형태에 대해 아무런 반응을 보이지 않는다는 말이다.

게다가 에러 코드는 말 그대로 선택 익스텐션의 일부일 뿐이다. 필자는 요 근래 타입 안정성에 대한 에러 코드나 필드나 리졸버에서 가능한 모든 에러 코드를 보여주는 GraphQL 툴을 본 적이 없다.

## 에러 배열을 사용하는 순간 다시 예전의 타입을 추론하는 예전 방식으로 돌아가게 된다.
백엔드와 프론트엔드 개발자들은 이제 초기에 GraphQL로 전환하기 이전에 피하고 싶었던 상황보다 더 고통스러운 순간에 직면하게 된다. 아무리 타입이 잘 갖춰진 GraphQL API라도 문서 하나쯤은 있어야 한다는 말이다. GraphiQL 이나 GraphQL Playground 와 같은 툴로 생성된 API 브라우저는 GraphQL API가 어떤 것들을 제공하는지 좀 더 쉬운 방법으로 볼 수 있게 해줘야겠지만 그렇다고 해당 툴들이 예시와 함께 제시하는 문서를 대체하진 않듯이 말이다.

## GraphQL 원시 타입을 사용하면 좀 더 나은 에러 핸들링을 할 수 있다.
최근에 에러 핸들링을 하기 위해 유니온 타입(union type)을 사용하는 것이 화두가 되었다. 유니온 타입은 해당 필드에서 반환 가능한 객체의 모음을 의미한다.

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

위 스키마에서는 `user` 필드에서 `User` 또는 `UserNotFoundError` 를 반환하는데 이는 리졸버 내에서 에러를 반횐하는 것 대신에 간단히 다른 타입을 반환할 수 있음을 의미한다. 이를 활용하면 서버에 보내는 쿼리 요청은 다음과 같게 된다.

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
위와 같이 요청을 하면 `apollo-server` 리졸버에서는 아래와 비슷하게 응답을 하게 된다.

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

유니온을 사용하게 되면 `__typename` 을 반드시 반환해줘야 하는데 이는 `apollo-server` 에서 어떤 타입의 결과 및 어떤 리졸버 타입의 필드 값이 리졸버 맵에서 사용되는지 알아야 하기 때문이다. 이렇게 하면 에러를 일반 GraphQL 타입처럼 사용할 수 있게 된다. 이 방법을 적용하면 메세지를 작성하고 에러코드를 보내는 대신 더 복잡한 타입을 다룰 수 있는 타입 안정성을 되찾게 된다.

아래 코드 예시는 `UserRegisterInvalidInputError` 에러 타입을 반환하는 로그인 뮤테이션이다. 일반적인 에러 메세지를 포함하고 있음에도 이 타입을 사용하면 단일 입력 필드를 사용할 수 있다.

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
여기에 신규 또는 좀 더 복잡한 `object` 타입 또한 추가하여 반환 할 수 있다. 클라이언트 단에서의 적용은 아래와 같다.

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

## GraphQL을 사용하면 UI에 맞는 데이터 트리를 제공 할 수 있다.
이것이 바로 UI에 맞는 에러 타입을 제공해야 하는 이유이다. 만약 다른 타입의 에러가 필요하다면 해당 에러에 맞는 타입을 새로 만들어서 유니온 리스트에 추가하면 된다.

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

위와 같이 구성하면 각 프로퍼티마다 에러타입을 추가 할 수 있게 된다. 이제 프론트엔드 쪽으로 넘어가서 해당 스키마의 요구사항을 살펴보도록 하자.

> X 국가의 괴상한 규제 때문에 해당 국가에 거주하는 사용자에게 더이상 회원가입을 하지 못하도록 변경하는 API 요구사항이 추가되었다.

겉보기엔 백엔드에 새로운 타입을 추가하기만 되는 것 처럼 보인다. 안타깝게도 이를 처리하려면 새롭게 추가된 에러 타입 때문에 프론트엔드 개발자또한 쿼리를 변경해야 한다.

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
위와 같은 쿼리를 아래와 같이 변경해야 한다.

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
이렇게 하지 않으면 클라이언트는 `CountryBlockedError` 에러를 받지 못하기 때문에(스키마에 존재하지 않음) 해당 에러를 표시하지 못한다.

개발자에게 새로운 에러타입을 추가 할 때마다 GraphQL 문서를 업데이트 하도록 강제하는 것은 그다지 현명한 선택이 아닌 것 처럼 느껴진다. 이제 우리가 새로 추가한 에러 객체를 한번 살펴보자.

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
새롭게 추가한 두 에러 타입 모두 공통적으로 `message`라는 속성(또는 필드) 가 추가된 것을 볼 수 있다. 게다가 우리는 유니온에 추가될 모든 에러들 또한 이 메세지 속성을 가질 것이라고 추측 할 수 있다. 다행히 GraphQL에서는 `interfaces`라는 것을 활용하여 추상화를 할 수 있도록 제공하고 있다.

```graphql
interface Error {
  message: String!
}
```
인터페이스는 각 타입마다 공통적으로 사용되는 필드를 제공하고 있다.

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

쿼리의 경우 인터페이스의 장점은 타입 대신 인터페이스를 통해 데이터 선택을 선언 할 수 있다는 점이다. 이는 곧 우리가 위에서 살펴보았던 스키마를 아래와 같이 변경가능하다는 것이다.

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
위와 같이 `UserRegisterInvalidInputError` 와 `CountryBlockedError` 모두 `Error` 인터페이스를 사용하여 적용 할 수 있다. 이제 아래와 같이 쿼리를 사용할 수 있게 되었다.

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
위와 같이 하면 `CountryBlockedError`을 선택 셋(set)에서 더이상 선언할 필요가 없어진다. `Error` 선택 셋에서 자동으로 커버된다. 추가로 `Error` 인터페이스로 상속받은 새로운 타입을 `UserRegisterResult` 유니온에 추가하게 되면 에러 메세지는 자동으로 해당 결과에 포함된다. 물론 클라이언트에서 해당 에러 상태를 핸들링 하기 위해서 로직 추가가 필요하지만 모든 에러를 `UserRegisterInvalidInputError` 와 같이 직관적으로 해결하는 것 대신 `CountryBlockedError`와 같이 특정 에러 다이얼로그만 표시하는 것도 가능하다.

모든 에러 타입을 `Error` 로 끝나도록 컨벤션을 구성하는 경우 여러 에러 타입을 포함하는 추상 에러 인터페이스를 만들어도 된다.

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

팀 내에서 새로운 에러 타입은 기존 에러들과 다르게 다루어져야 한다고 결정을 내린다면 간단하게 `useEffect`에 새로운 조건문을 추가하기만 하면 된다.

Original Source:
[Handling GraphQL errors like a champ with unions and interfaces](https://blog.logrocket.com/handling-graphql-errors-like-a-champ-with-unions-and-interfaces)