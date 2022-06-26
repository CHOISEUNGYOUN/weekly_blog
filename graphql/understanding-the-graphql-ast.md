# GraphQL 추상 구문 트리(AST) 이해하기
GraphQL을 사용하면 프론트엔드 엔지니어들이 엄격한 백엔드 응답 방식을 따르지 않고도 원하는 대로 데이터를 호출할 수 있다.
Eric Bauer 가 [‘The Evolution of API design’](https://www.youtube.com/watch?v=PmWho45WmQY) 에서 언급했듯이 GraphQL을 사용하면 자연스럽게 데이터를 표현할 수 있게 되기 때문에 호출단에서 유연하게 데이터를 다룰 수 있다. 이런 독특하고 복잡한 요청을 해석하고 처리하기 위해서 GraphQL에서는 추상 구문 트리(Abstract Syntax Tree, AST)를 이용하여 요청을 구성한다. 이 AST를 사용하면 요청하는 데이터를 백엔드에서 손쉽게 파싱하고 구성할 수 있다.

GraphQL은 단지 사양일 뿐이고 최종 데이터 소스와는 무관하다. 이로 인해 [graphql-js](https://github.com/graphql/graphql-js) 나 [graphql-tools](https://github.com/ardatan/graphql-tools) 과 같은 라이브러리를 활용하여 추상 구문 트리를 직접 받지 않고 추상화 하는 작업이 선행된다. 그럼에도 종종 특정 사용자 요청에 따라 커스텀 동작 구문을 만들어 사용하게 된다.

## 추상 구문 트리는 무엇인가?
Stephen Schneider 가 [‘The GraphQL AST— Dawn of a schema’](https://www.youtube.com/watch?v=sbOe6fU0SCI)에서 언급했듯이 AST는 '단지 중첩 객체들을 이쁘게 잘 정돈하는 방법' 이라고 볼 수 있다. 이는 컴파일러에서 작성한 코드를 구문 분석하여 프로그래밍 방식으로 탐색할 수 있는 트리 구조로 변환하는 데 일반적으로 사용되는 구조이다.

사용자가 요청을 하는 경우 GraphQL은 사용자가 요청한 쿼리문을 직접 정의한 스키마 정의와 결합한다. 이 때 스키마 정의는 추상 구문 트리 형태로 저장된 리졸버에 대한 스키마 정의이다. 이 추상 구문 트리는 어떤 필드들이 요청되었는지, 어떤 인자들이 포함되어있는지 등등을 판단하기 위해 사용된다.

럭비 선수 리스트를 백엔드에 요청한다고 가정해보자. 이 경우 스키마 정의는 아래와 같을 것이다.

```js
export const RugbyPlayer = new gql.GraphQLObjectType({
    name: 'RugbyPlayer',
    fields: {
        full_name: {
            type: gql.GraphQLString,
        },
        club: {
            type: new gql.GraphQLObjectType({
                name: 'Club',
                fields: {
                    name: {
                        type: gql.GraphQLString,
                    },
                },
            }),
        },
    },
})
```

사용자가 목록을 요청 할 때 GraphQL 스키마는 해당 쿼리문에 대한 정보가 없다. 결국 해당 요청은 추상 구문 트리 형식으로 구문 처리되어 요청에 대한 검증(부정확한 필드 요청이나 인자에 대한 에러 처리)을 하게 된다. 검증이 한번 일어나게 되면 스키마 타입은 요청에 해당되는 '가지'에 매핑되어 더 유용한 메타데이터를 제공하게 된다.

사용자가 럭비 선수들의 리스트와 어떤 팀에 소속되어 있는지 데이터를 요청한다고 가정해보자. 이 경우 쿼리문은 아래와 같을 것이다.

```js
export const RugbyPlayers : gql.GraphQLFieldConfig<any, any> = {
    description: 'A list of rugby players.',
    type: new gql.GraphQLList(RugbyPlayer),
    resolve: (parent, args, context, info) => ([{
        full_name: 'Matthew Phillip',
        club: {
            name: 'Melbourne Rebels',
        },
    }, {
        full_name: 'Israel Folau',
        club: {
            name: 'NSW Waratahs',
        },
    }]),
}
```

* `부모(parent)` - 루트(root)로도 알려져 있다. 리졸버는 다른 리졸버에 해당 인자가 이전 요청의 결과를 가지고 있는지 여부를 위임 할 수 있다.
* `args` — 사용자가 요청한 매개변수 값이 들어있다.
* `context` — 리졸버 체인으로 전달되는 전역변수. 다른 계층과 상호 통신을 해야하는 경우 유용하다.
* `info` — 추상 구문 트리에 대한 정보

생성된 추상 구문 트리 쿼리문은 아래와 같다

```json
{  
   "fieldName":"RugbyPlayers",
   "fieldNodes":[  
      {  
         "kind":"Field",
         "name":{  
            "kind":"Name",
            "value":"RugbyPlayers",
         },
         "arguments":[ ],
         "directives":[],
         "selectionSet":{  
            "kind":"SelectionSet",
            "selections":[  
               {  
                  "kind":"Field",
                  "name":{  
                     "kind":"Name",
                     "value":"full_name",
                  },
               },
               {  
                  "kind":"Field",
                  "name":{  
                     "kind":"Name",
                     "value":"club",
                  },
                  "selectionSet":{  
                     "kind":"SelectionSet",
                     "selections":[  
                        {  
                           "kind":"Field",
                           "name":{  
                              "kind":"Name",
                              "value":"name",
                           },
                        }
                     ],
                  },
              },
          },
      }
   ],
   "returnType":"[RugbyPlayer]",
   "parentType":"Query",
   "path":{  
      "key":"RugbyPlayers"
   },
   "schema":{...},
   "fragments":{  },
   "operation":{...},
   "variableValues":{}
}
```

* `fieldName`: 지금 요청한 리졸버 이름
* `fieldNodes`: selectionSet(트리에서 탐색 중인 현재 객체의 필드 그룹) 의 필드 배열
* `path`: 현재 분기로 이어지는 모든 상위 필드를 추적
* `operation`: 모든 쿼리의 정보. (이 정보는 모든 리졸버에 동일하게 적용되는 글로벌 필드. 다른 글로벌 필드에는 schema, fragments, rootValueand variableValues 등이 있다.)

추상 구문 트리의 구조는 GraphQL의 중첩 객체로 이루어진 요청을 다루는데 최적화 되어있다. 위에서 언급된 추상 구문 트리는 루트의 `full_name` 필드와 중첩된 `club` 필드를 깔끔하게 구분하고 있다. `selectionSet`을 통해 `club` 또한 사용자가 요청한 속성이라는 사실을 알 수 있다. graphql-tools 과 같은 툴들은 이런 selection sets 검색 및 추상 구문 트리를 다루는데 용이하다.

## So why is all this important?
When I first starting implementing a GraphQL backend, my thoughts were not “I can’t wait until I can dive into that confusing GraphQL AST that I keep seeing scattered throughout my resolvers”.

No.

Instead it arose out of need to craft custom directives and optimise user requests. It has been very useful to break the traditional GraphQL lifecycle and intercept a request before it gets passed onto another library to generate a data response. Specifically, by traversing and augmenting the AST we can implement:

* Schema stitching
* Custom directives
* Enriched queries
* Layered Abstraction
* More backend magic!

## Custom Directives
In a step to give more control to the front-end client, we often find it useful to implement a range of directives that can transform or filter out fields specified in the AST.

It is often very common that rugby commentators mis-pronounce the players name. To help our faithful commentators, we have decided to implement a rugby player **directive** to help with pronunciations.

To achieve the above, we can create a `@pronounce` directive that will give us the phonetic spelling of players names.

```js
export default new gql.GraphQLDirective({
    // register the directive name
    name: 'pronounce',
    description: 'Converts a string to the phonetic pronounciation',
    locations: [
        // We only allow this on fields
        gql.DirectiveLocation.FIELD,
    ],
})

// In order to use the directive, we must also register the directive on our schema
const root = new GraphQLSchema({
    baseSchema,
    directives: [
        filter,
    ],
})
```


A simple implementation of a magic service can then help us ‘translate’ these players names by diving into the `directives` array on the `full_name` selection in the generated AST.


Our resolve function now dives into the AST to find which fields are tagged with the `@pronounce` directive and translates the fields accordingly.

```js
// For demonstration purpose, we are only diving 1 level deep into AST
const fieldsToTranslate = info.fieldNodes[0].selectionSet.selections
    // Find any fields (nodes) with the @pronounce directive
    .filter(selection => !!selection.directives.find(directive => directive.name.value === 'pronounce'))
    .map(selection => selection.name.value)

// Map through each field and translate if needed
return rugbyPlayers.map(player => Object.keys(player).reduce((obj, field) => {
    const doTranslate = fieldsToTranslate.includes(field)

    const value = doTranslate
        // Call magic machine learning service to create phonetic spelling
        ? translate(player[field])
        : player[field]

    return {
        ...obj,
        [field]: value,
    }
}, {}))
```

With a few lines of code we have solved the issue of dodgy name pronunciations.

## Caching
What nightmares are made of.

Many objects in a GraphQL data source do not change that often. It becomes quite expensive to fetch these items every time a user requests it and since users define the shape of a GraphQL response, it can be quite difficult to cache and return a custom response for each user.

A common solution to this is caching a result against **a unique stringified version of the AST field selections**. By combining the requested fields and their nested fields, we can create a key that will match for all users with the same query document. This enables us to reuse a result instead of fetching from a database or external data source once again.

## Conclusion
Understanding how the AST is structured and used within your resolvers allows you to gain a fine-grain control over the input and output of your GraphQL backend. It allows you to cache results, create custom directives, optimise queries & stitch together data from multiple resources.

By understanding how the AST works in the GraphQL lifecycle, you can create complex backends and ultimately provide more power to the front end consumer.

Original Source:
[Understanding the GraphQL AST](https://adamhannigan81.medium.com/understanding-the-graphql-ast-f7f7b8e62aa4)