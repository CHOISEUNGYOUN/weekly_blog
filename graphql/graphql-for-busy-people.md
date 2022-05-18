# GraphQL 라이프사이클 이야기

최근에 GraphQL을 사용하는 프로젝트를 진행하게 되었다. 이 프로젝트를 시작하기 전에 어떻게 쿼리가 동작하는지 이해하고자 하였다. GraphQL의 요청 라이프사이클을 설명해주는 다이어그램을 열심히 찾아보았지만 요구사항을 충족하는 내용이 어디에도 없었다. 그리하여 GraphQL이 어떻게 동작하는지 이해할 수 있는 다이어그램을 직접 만들게 되었다.

![img](/graphql/imgs/graphql-for-busy-people-1.png "GraphQL Request Lifecycle")

위 다이어그램을 설명하기 위해 `cities`라는 데이터베이스를 만들었다. 사용자는 도시들을 언어별로 검색할 수 있고 해당 언어 기반의 다른 사용자의 리뷰들을 볼 수 있도록 구현했다. 해당 도시의 리뷰에는 리뷰를 남긴 사용자, 5점 척도의 리뷰 점수, 그리고 추가 코멘트를 남길 수 있게 구성되어있다.

아래 예시는 실제 GraphQL에서 `Arturan` 언어로 검색 요청하는 쿼리이다.

```gql
query {
  cities(language: "Aturan") {
    id
    name
    language
    reviews {
      adventurer {
        name
      }
      rating
      comment
    }
  }
}
```
응답값은 아래와 같다.

```json
{
  "data": {
    "cities": [
      {
        "id": "1",
        "language": "Aturan",
        "name": "Imre",
        "reviews": [
          {
            "adventurer": {
              "name": "Kvothe"
            },
            "rating": 5,
            "comment": "Check out The Eolian for some good music"
          }
          ...
        ]
      }
      ...
    ]
  }
}
```

GraphQL의 공식문서에서는 응답값에 최상위 데이터 키가 있음을 보장하지만 그렇지 않은 경우 응답값이 요청값의 형태와 유사함을 알 수 있다. 이렇게 빠르고 유연하게 요청값을 구성할 수 있다는 점이 GraphQL의 장점이다

-----------------------------------------------------------------------------------------------------------

그렇다면 요청값이 어떻게 응답값으로 내려오게 되는 걸까?

## 1. 클라이언트에서 POST로 body에 쿼리를 요청

GraphQL 자체가 어느 특정 프로토콜에 종속되는 것은 아니지만 웹 어플리케이션에서는 보통 클라이언트 서버간 통신으로 HTTP를 사용한다. 특히 모든 요청을 POST 메소드를 통하여 이뤄지기 때문에 정보를 요청하든(GraphQL에서는 이를 "쿼리"라고 함) 데이터를 변경하든 POST만 사용하여 통신한다. 물론 GET 요청을 사용하여 쿼리 파라미터를 넘겨줄 수 있으나 이는 흔한 방식이 아니다.

쿼리를 실행하게 되면 클라이언트에서는 하나의 라우트로 POST 메세지를 보내게 된다. GraphQL에서는 보통 `/graphql`을 사용한다. 클라이언트 쪽의 요청을 단순화 하기 위해 Apollo 나 Relay와 같은 GraphQL 클라이언트를 사용한다. GraphQL 클라이언트는 프론트엔드 프레임워크 통합과 효율적인 캐싱 및 요청 관리와 같은 기능을 서버와 클라이언트에 제공하기 위해 사용되는 추가적인 레이어다.

## 2. 서버에서 요청 처리 및 검증

클라이언트에서 요청을 보낸 이후 서버는 해당 바디를 [추상 구문 트리(AST)](https://en.wikipedia.org/wiki/Abstract_syntax_tree)기반 쿼리를 생성하여 처리를 한다. 추상 구문 트리는 트리 형태의 데이터 구조로써 코드 비트의 중요한 구문 요소를 표시하는데 사용된다. GraphQL의 추상 구문 트리는 타입 및 값, 서버에서 요청을 검증하고 작업할 수 있게 하는 기타 정보가 담겨있는 쿼리를 인코딩한다.

GraphQL의 추상 구문 트리의 구조는 다음과 같다.

```json
{
  "kind": "Document",
  "definitions": [
    {
      "kind": "OperationDefinition",
      "operation": "query",
      "variableDefinitions": [],
      "directives": [],
      "selectionSet": {
        "kind": "SelectionSet",
        "selections": [
          {
            "kind": "Field",
            "name": {
              "kind": "Name",
              "value": "cities",
              "loc": {
                "start": 139,
                "end": 145
              }
            } ...
```

추상 구문 트리를 생성한 뒤 서버는 정의된 스키마 기반으로 요청을 검증한다. 이 검증 절차는 자동으로 수행되기 때문에 허용되지 않는 변수를 확인하는 등의 추가적인 코드작업이 필요없게된다.

GraphQL이 이러한 작업을 할 수 있는 이유는 타입과 타입간 관계를 정의 하기 위해 스키마 정의 언어(Schema Definition Language, SDL)를 채택하고 있기 때문이다. 위 쿼리 예시에서 볼 수 있듯이 우리는 사용자, 도시 및 그에 대한 리뷰 데이터를 요청 할 것이다.

아래 예시는 위 요청을 받기 위해 구성된 [GraphQL의 스키마 정의](https://graphql.org/learn/schema/#type-language)이다.

```gql
type City {
  id: Int!
  language: String!
  name: String!
  reviews: [Review!]
}
type Review {
  id: Int!
  rating: Int!
  comment: String
  adventurer: Adventurer!
}
type Adventurer {
  id: Int!
  name: String!
}
```

Apollo와 같은 GraphQL 클라이언트는 Scala, Swift 또는 TypeScript와 같은 다른 여러 언어들을 위한 타입 파일을 생성한다. 이 타입 파일을 생성하기 위해서 Apollo는 내부 구성 확인(instrospection) 쿼리를 사용한다. 최상위 단계에서는 해당 내부 구성 확인 쿼리들은 API 자체를 불러오기 위한 쿼리가 된다.

한번도 호출해보지 못한 API를 작업하게 되는 경우 내부 구성 확인 쿼리를 사용하여 API에서 허용하는 쿼리와 데이터 속성을 확인할 수 있다. GraphQL에서는 [GraphiQL](https://github.com/graphql/graphiql)이라고 불리는 인브라우저(in-browser) IDE를 사용하여 최신화된 문서 관리 및 속성 확인을 위한 내부 구성 확인 쿼리를 실행 해볼 수 있다.

## 3. 서버에서 쿼리 실행

쿼리가 검증된 이후에는 서버에서 해당 정보를 가져와서 요청 정보에 맞게 결과를 구성한다. 이때 각 쿼리 필드에 대한 리졸버 함수를 실행하게 된다.

각각의 리졸버들 또한 필터링 또는 페이지네이션 변수와 같이 쿼리에 전달된 모든 인자들을 처리한다. 해당 인자들 또한 속성 검사 및 적절한 리졸버에서 실행되었는지 여부를 확인하는 등 쿼리에 사용되는 필드들과 비슷하게 다루어진다.

Our root level query is cities, so we will initially execute the resolver for cities, and then move downward, executing resolvers for each field in the schema definition. The execution will wait until all the resolvers are finished, at which point the data will be returned in the response.

As you might imagine, we could easily run into situations where we are querying duplicate data, like if we fetched the same adventurer from the database multiple times. There are many strategies to avoid making duplicate requests, including caching and batching. Additionally, in order to avoid infinite loops or overly complex queries (like ones that are deeply nested or excessively large), depth limits and field constraints can be set for each query.

## 4. The client receives the response and updates client-side state.

At this point, our request has been validated and resolved by the server, and only the information the client asked for, with no extraneous fields or associations, is returned.

This is one of the major benefits of GraphQL: data management on the front-end can be moved to the component level. Each front-end component can decide what data it needs locally, create the appropriate query snippet, pass that to a GraphQL client, and then manage its data and state locally. Pushing down the data management to the component level has replaced application state management libraries like Redux.


Original Source:
[There and Back Again, A GraphQL Lifecycle Tale](https://thoughtbot.com/blog/graphql-for-busy-people)
