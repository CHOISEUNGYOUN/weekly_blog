종종 Ruby 프로그램은 네트워크 타임아웃 에러와 같이 우리가 100% 예상치 못한 에러들을 발생시킨다. 이러한 에러들 또한 예외처리 해줘야 하는데, 이를 위해 다양한 `Exception` 클래스들을 다뤄야 한다. 이러한 클래스들이 어떻게 세세하게 나누어져 있는지 알아보자.

## 예외처리 되는 클래스는 반드시 `Exception` 클래스로부터 파생되어야 한다.
Ruby의 `Exception` 클래스의 층위는 `Exception` 으로 부터 시작된다. 만약 `Exception` 클래스가 아닌 클래스를 `raise` 처리를 하려고 한다면, 다음과 같은 에러를 만나게 된다.

```rb
begin
  raise 1234.0
rescue => error
  puts error.inspect
end

#<TypeError: exception class/object expected>
```

## 기본값은 `StandardError` 로 지정되어 있다.

`Exception`에 속하는 하위 에러 클래스를 따로 명시해주지 않으면, Ruby는 기본적으로 `StandardError` 에 속하는 에러 클래스들을 호출한다. `StandardError`가 아닌 다른 에러 클래스를 예외처리 하고 싶다면 다음과 같이 명시 해줘야한다.


```rb
begin
  raise Exception.new
rescue Exception => error
  puts "Correct!"
end

#Correct!
```

## `Exception`에 속한 클래스들을 예외처리 하는 것은 통상적인 방법이 아니다.

우리는 모든 에러들을 예외처리 하려고 하지 않는다. 에러 중에는 우리가 `command + c` 를 누를 때 발생하는 `NoMemoryError` 와 같이 `StandardError`에 속하지 않은 에러들도 있다. 일반적으로 Ruby 프로그램에서 말하는 예외처리는 `StandardError` 에 속한 에러들을 처리한다는 의미다.

## 최선의 시나리오

예외 처리를 하는 가장 좋은 시나리오는 어떤 에러들이 발생할 것인지 예상하고 100% 사전에 방지를 할 수 없는 에러 클래스들을 예외처리 하는 것이다.

```rb
HTTP_ERRORS = [
  EOFError,
  Errno::ECONNRESET,
  Errno::EINVAL,
  Net::HTTPBadResponse,
  Net::HTTPHeaderSyntaxError,
  Net::ProtocolError,
  Timeout::Error
]

begin
  some.http.call
rescue *HTTP_ERRORS => error
  notify_airbrake(error)
end
```

위 예시와 같이 예외처리 할 특정 클래스들만 지정함으로써 우리가 미처 예상하지 못했던 에러들을 예외처리 하고 `nil` 을 리턴하는 등의 실수를 방지 할 수 있다.

## 어쩔 수 없는 경우...

가끔 어떠한 에러가 발생하는지 예측 하지 못하는 경우가 있는데, 이러한 경우에는 `StandardError` 를 명시해준다.

```rb
begin
  some.unique.situation
rescue StandardError => error
  notify_airbrake(error)
end
```

Original Source:
[Rescue StandardError, Not Exception](https://thoughtbot.com/blog/rescue-standarderror-not-exception#the-rescued-class-must-descend-from-exception)
