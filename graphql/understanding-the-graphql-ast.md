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

## 그럼 앞서 언급한 내용들이 왜 중요할까?
처음 GraphQL 백엔드를 구축하기 시작했을땐 GraphQL의 복잡한 추상 구문 트리를 이해하여 구현된 리졸버에 파편화 된 요소들을 바로 잡을 수 있을 것이라고 생각했다. 이내 잘못된 생각을 하고 있었다고 깨닫게 되었다. 대신 커스텀 지시문을 만들어서 사용자 요청을 최적화 시켜야 한다는 생각이 들었다. 전통적인 GraphQL 라이프사이클을 파헤친 뒤 요청 중간에 내가 원하는 지시문을 추가하는데 매우 유용했다. 특히 아래와 같은 작업들을 추상 구문 트리에 적용할 수 있었다.

* Schema stitching(여러 개의 GraphiQL 모델을 하나로 통합하는 기법)
* Custom directives(커스텀 지시어, 스키마에 @deprecated 지시어를 넣는것 처럼 직접 커스텀으로 지시어를 추가할 수 있음.)
* Enriched queries(쿼리 형태를 다양하게 사용할 수 있음을 의미하는 듯.)
* Layered Abstraction(스키마는 추상 구문 트리를 제외한 실제 필요한 타입과 리졸버만 보여준다/)
* 이외에도 여러가지 마법같은 방법들이 있다!

## 커스텀 지시어
프론트엔드 클라이언트에서 API를 좀 더 유연하게 사용하기 위해 추상 구문 트리에 지정된 필드를 변환하거나 필터링 할 수 있는 다양한 지시어를 사용할 수 있다.

예를 들어 럭비 평론가들은 선수들 이름을 잘못 발음하기 일쑤이다. 이런 평론가들의 실수를 줄여주기 위해 럭비선수 타입에 발음표기를 커스텀 지시어로 추가 할 수 있다. 이를 위해 `@pronounce`라는 지시어를 만들어 각 선수들의 발음 표기를 제공 해 줄 수 있다.

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

추상 구문 트리 내 `full_name` 선택자에 선수에 대한 발음을 지시어로 간단히 추가함으로써 럭비 선수들의 이름을 어떻게 말하는지 알수 있게 되었다.

리졸버 함수는 이제 추상 구문 트리 내 어떤 필드에 `@pronounce` 지시어가 포함되어있는지 찾아 그에 맞게 변환시켜준다.

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

몇줄의 코드 추가만으로 선수들의 이름을 제대로 발음하지 못하는 문제를 해결 할 수 있다.

## 캐싱
GraphQL에서 가장 어려운 부분이 캐싱이다. GraphQL 데이터 소스 내 많은 객체들은 그리 자주 변경되지 않는다. 사용자가 요청할 때마다 이에 상응하는 데이터를 가져오는 것은 꽤나 비효율적이다. 이는 사용자가 GraphQL 응답 필드를 지정하기 때문에 캐싱하기가 어렵고 매 사용자마다 고유의 응답을 내려주기 힘들기 때문이다.

이에 대한 일반적인 해결책은 **문자열의 추상 구문 트리 선택자에 대한 결과 자체를 캐싱** 하는 것이다. 요청되는 필드와 중첩 필드들을 결합하게 되면 동일한 쿼리문을 요청하는 사용자를 구분 할 수 있는 키를 생성할 수 있게 된다. 이를 통해 데이터베이스에서 데이터를 다시 조회하거나 외부에서 데이터 소스를 가져오는 것이 아니라 결과값을 재사용 할 수 있게 된다.

## 결론
추상 구문 트리가 어떻게 구성되고 리졸버에서 어떻게 사용되는지 이해하게 되면 GraphQL 백엔드에서 입력과 출력값을 세밀하게 조정할 수 있게 된다. 이를 통해 결과를 캐싱할수도 있고, 커스텀 지시어를 생성할 수도 있으며, 다양한 소스에서 가져온 데이터를 쿼리 최적화 및 변환을 할 수 있다.

추상 구문 트리의 라이프사이클을 이해함으로써 좀 더 복합적인 백엔드를 구성하여 최종적으로 프론트엔드에게 좀 더 강력한 힘을 제공할 수 있다.

Original Source:
[Understanding the GraphQL AST](https://adamhannigan81.medium.com/understanding-the-graphql-ast-f7f7b8e62aa4)