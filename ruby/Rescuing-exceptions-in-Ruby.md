예외 처리를 하는 것은 콜스택에서 어떤 작업이 처리 될때 전체 어플리케이션이 비정상적으로 중단되는 현상으로부터 보호 할 수 있다. Ruby 에서는 `rescue` 키워드를 예외 처리할 때 사용한다.

Ruby 에서 예외처리 할 때, 에러 클래스를 특정하여 사용 할 수 있다.

```rb
begin
  raise 'This exception will be rescued!'
rescue StandardError => e
  puts "Rescued: #{e.inspect}"
end
# 참고: raise 키워드 사용 할 때 에러 클래스를 명시 해주지 않으면, 루비는 기본값으로 RuntimeError 처리를 할 것이다.
```

하나의 클래스를 처리 하는것 뿐만 아니라 여러 종류의 클래스 또한 예외처리 가능하다. 이 경우에는 여러 에러들이 동일하게 처리 된다.

```rb
begin
  raise 'This exception will be rescued!'
rescue StandardError, AnotherError => e
  puts "Rescued: #{e.inspect}"
end
```
각각의 에러를 다르게 처리하고 싶으면 각 에러 클래스에 대한 `rescue` 를 작성하여 처리 할 수 있다. 이러한 기능은 라이브러리에서 각 케이스마다 다른 예외처리를 해줄 때 유용하다.

```rb
begin
  raise 'This exception will be rescued!'
rescue StandardError => e
  puts "Rescued: #{e.inspect}"
rescue AnotherError => e
  puts "Rescued, but with a different block: #{e.inspect}"
end
```

### 예외 클래스 계층
Ruby는 타입 종류별로 에러를 구분하기 위해서 예외 케이스들을 계층으로 나누어 구분한다. 이는 각 에러 클래스에 대한 블록처리 할 필요없이 특정 상위 계층의 클래스를 함으로써 동일한 기능을 구현 할 수 있게 했다.

각 라이브러리 마다 고유의 에러 클래스를 만들 수 있지만, Ruby 2.5 버전의 기본 내장 에러 클래스들은 다음과 같다.

```rb
- NoMemoryError
- ScriptError
    - LoadError
    - NotImplementedError
    - SyntaxError
- SecurityError
- SignalException
    - Interrupt
- StandardError (default for `rescue`)
    - ArgumentError
        - UncaughtThrowError
    - EncodingError
    - FiberError
    - IOError
        - EOFError
    - IndexError
        - KeyError
        - StopIteration
    - LocalJumpError
    - NameError
        - NoMethodError
    - RangeError
        - FloatDomainError
    - RegexpError
    - RuntimeError (default for `raise`)
    - SystemCallError
        - Errno::*
    - ThreadError
    - TypeError
    - ZeroDivisionError
- SystemExit
- SystemStackError
- fatal (impossible to rescue)
```

`rescue` 블록에서 에러 클래스를 명시 하지 않으면 루비는 기본값인 `StandardError` 을 인식한다. 이 경우에는 `StandardError` 의 하위 클래스인 `ArgumentError` 와 `NoMethodError` 가 예외처리 될 것이다.

예외 계층이 어떻게 작동하는지 가장 잘 보여주는 예는 low-level 플랫폼 의존적 예외 클래스인 `SystemCallError` 다. 이 에러는 주로 파일 읽기/쓰기 작업할 때 발생한다.

Ruby 에서 `File.read` 메소드가 파일 읽기를 실패한 경우 예외 처리를 한다. 이는 파일이 존재하지 않거나 해당 프로그램이 읽기 권한이 없는 등 여러가지 이유로 발생 할 수 있다.

이러한 문제는 플랫폼 의존적이기 때문에 Ruby 는 어떤 OS 기반으로 운영되느냐에 따라 다른 예외처리를 수행한다. Ruby 에서 이러한 low-level 에러들은 각 플랫폼 마다 다른 종류의 `Errno::*` 예외처리를 실행한다.

모든 `Errno::*` 예외 클래스는 `SystemCallError` 의 하위 클래스이다. 에러들이 플랫폼 별로 상이하지만, `rescue` 블럭에 `SystemCallError` 를 표시하여 예외처리를 할 수 있다.

```rb
begin
  File.read("does/not/exist")
rescue SystemCallError => e
  puts "Rescued: #{e.inspect}"
end
```

### 예외 케이스 숨기기 (Swallowing exceptions)

예외 처리 할 시, 의도치 않게 다른 예외 케이스들을 함께 처리하지 않도록 어떤 예외 클래스를 다룰 것인지 명확하게 지정해야 한다.

```rb
image = nil

begin
  File.read(image.filename)
rescue
  puts "File can't be read!"
end
```

위 예시에서 `image` 변수의 값은 `nil` 이다. 따라서 해당 변수의 파일명을 읽으려고 시도 하면 루비는 `NoMethodError: undefined method 'filename' for nil:NilClass` 호출 할 것이다. 하지만 `rescue` 블록에 에러 클래스를 지정하지 않았기에 `StandardError` 에 해당되는 모든 하위 에러 클래스들이 예외처리(`NoMethodError` 포함) 될 것이고 "File can't be read!" 메세지를 출력한다. 이는 해당 코드의 잠재적 버그를 숨기는 것과 같다.

***참고: 이런식으로 구현 할 수 있으나, 최상위 에러클래스(Ruby 에서는 `StandardError`) 를 예외 처리하는 것은 지양 할 것을 권장한다.***

Original Source:
[Rescuing exceptions in Ruby](https://blog.appsignal.com/2018/04/10/rescuing-exceptions-in-ruby.html)
