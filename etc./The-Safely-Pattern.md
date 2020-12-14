`Safely Pattern` 은 간단히 말하자면 중요하지 않은 기능을 담당하는 코드를 감싸는 역할을 한다. 에러 예외 처리 최상단에 위치하며 다음과 같은 규칙을 따른다.
1. 개발 환경이나 테스트 환경에서는 에러를 리턴한다.
2. 다른 환경(예: 배포 환경) 에서는 예외 처리하며 해당 에러 로그를 남긴다.

아래 예제코드는 Javascript 로 구현한 간단한 `Safely Pattern` 이다.

```js
function safely(nonCriticalCode) {
  try {
    nonCriticalCode();
  } catch (e) {
    if (env === "development" || env === "test") {
      throw(e);
    }
    report(e);
  }
}
```

보통의 예외처리와는 다르게 다음과 같은 이점이 있다.
1. 개발 환경이나 테스트 환경에서 처리하지 못한 에러들을 쉽게 알아차리고 디버깅 할 수 있게 해준다.
2. [`DRY (Don't Repeeat Yourself)`](https://en.wikipedia.org/wiki/Don't_repeat_yourself) 룰에 따른 로깅을 할 수 있게 해준다.

예외 처리에 따른 특정 표식을 남기는 것을 권장 하는데, 이렇게 하면 에러 로그가 `Safely Pattern` 에 따라 처리 되었다는 것을 쉽게 알 수 있다. 대표적인 표기법으로 에러 로그에 `[Safely]` 접두어를 남기는 것이다.

해당 패턴은 Ruby 와 Javascript 에서 많이 쓰이고 있다.

Original Source:
[The Safely Pattern](https://ankane.org/safely-pattern)
