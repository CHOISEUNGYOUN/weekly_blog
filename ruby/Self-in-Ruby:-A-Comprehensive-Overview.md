## Ruby의 `self`에 관한 개론

많은 프로그래밍 언어에서 `self` 또는 `this` 키워드는 개발자에게 특정 클래스나 오브젝트의 컨택스트를 참조 할 수 있게 해주는 강력한 도구이다. 어떠한 컨텍스트에 상관없이 현재 가리키는 오브젝트를 참조하는 기능은 루비와 상관없이 어떠한 언어에서도 100% 완벽하게 이해하기 어려울 수 있다.<br/>

이 글에서는 루비 및 루비 내에서 보편적으로 사용되고 있는 메소드에서 `self` 라는 키워드가 정확하게 무엇을 의미하고 있는지 알아보고자 한다. 이를 통해서 우리는 루비 내에서(또는 기타 다른 객체지향 패러다임 언어들) 에서 `self`가 참조되는 키워드들이 어떤식으로 다뤄지는지 이해 할 수 있을 것이다. 함께 파고들어보자!

## 컨텍스트 내에서의 `self`
루비 내에서 `self` 의 목적(이에 따른 동작 방식)은 상황별 또는 어떠한 컨텍스트 내에서 사용되느냐에 따라 갈린다. 다시 말하자면, 어떤 경우에는 컨텍스트 내에서는 부모 클래스가 될수도 있고, 다른 경우에는 특정 인스턴스의 부모 클래스를 가리킬수 있다. 이런 다양한 컨텍스트들을 이해하기 위해 예제 코드를 통하여 하나하나 짚어보고자 한다. <br/>

어떤 컨텍스트에 상관없이 `self`의 중요한 한가지 속성은 한번에 하나의 오브젝트만 가리킨다는 점이다. 이러한 특성은 현재 주어진 컨텍스트 내에서 `self`가 어떤 특정 오브젝트를 가리킬지 판단 할 수 있게 해줍니다. 적절한 `self`사용은 대다수의 `Ruby` 프로젝트에서 개발 효율성을 가져다 줄 것이다.

## 최상위 컨텍스트 내에서의 `Self`
루비에서의 최상위 컨텍스트는 어떠한 클래스, 모듈, 메소드 또는 기타 하위레벨 에서 실행되지 않는 코드를 말한다. 전형적인 예로 간단한 스크립트나 규모 있는 앱에서 객체지향 컴포넌트 없이 `.rb` 파일 내에서 바로 실행되는 코드를 들 수 있다. <br/>

한 예로 여기 연속적으로 실행되는 4줄의 간단한 스크립트가 있다. 이 코드는 `name` 변수를 최상위 컨텍스트에서 실행되고, 이어서 3줄의 `puts` 명령으로 `name` 변수와 `self`를 콘솔에 출력한다.

```rb
# top-level.rb
name = "Jane"
puts "Name is: #{name}"
puts "Self is: #{self}"
puts "Self class is: #{self.class}"
```

출력은 다음과 같다.


```shell
Name is: Jane
Self is: main
Self class is: Object
```

여기서 중요한 점은 루비 내에서 최상위 컨텍스트에서 `self`가 호출될 때, `main` 오브젝트를 호출 한다는 것이다. 이는 루비 내에서 작성된 모든 코드는 `main` 오브젝트 컨텍스트 내에서 실행되기 때문이다. 특정 클래스 내에서 실행 할 시, 부모 클래스가 컨텍스트임은 당연하나, 루비 코드가 처음 실행 될 때 `main` 오브젝트를 자동으로 생성한다. 그러므로 상기 예제코드에서 가리키는 `self`는 루비가 처음 실행 될때 생성하는 `main` 오브젝트 인스턴스가 된다.<br/>

## 클래스 내에서 `Self` 정의
루비에서 클래스를 정의 할 때, `self`는 해당 컨텍스트를 참조 할 수 있다. 아래에 `Author`라는 클래스를 정의하고 그 내부에 `self`를 출력하는 예제코드를 살펴보자.

```rb
# class-definition.rb
class Author
    puts "Self is: #{self}"
end
```

여기서 `self`는 클래스 정의의 내부 즉, `self`가 호출된 부모 클래스 `Author`와 동일하다. 사실 위와 동일한 컨텍스트에서는 `self`가 실제 `Author` 클래스의 이름의 대체라고 여겨질수도 있다. `self` 대신에 `Author`를 직접 호출할 수도 있으며, 루비는 100% 동일하게 취급 할 것이다.

```shell
Self is: Author
```
<br/>

## 모듈 내에서 `Self` 정의

모듈 정의 내부에서의 `self` 사용은 클래스 정의와 매우 흡사하다. 사실 루비 언어 안에서 사용되는 `self`는 클래스 컨텍스트 내부이든 모듈 컨텍스트 내부이든 기본적으로 동일하게 처리된다.

예를 들어 `Library`라는 모듈 안에 `Author`라는 클래스를 선언하여 배치한다고 보면, `self`의 출력값은 호출되는 `self`의 컨텍스트가 된다.

```rb
# module-definition.rb
module Library
    puts "Self inside Library is: #{self}"

    class Author
        puts "Self inside Library::Author is: #{self}"
    end
end
```

루비는 클래스 정의에서 그랬던것 처럼, 모듈 정의에서 호출되는 `self`또한 해당 컨텍스트의 부모 레벨인 `Library`를 호출한다. 여기서 `Author` 클래스는 부모로 `Library` 모듈을 가지고 있으며, 루비는 이것을 인식하여 해당 층위 또한 읽어온다. 그리하여 우리는 모듈에 속한 클래스를 가져올 수 있다.

```shell
Self inside Library is: Library
Self inside Library::Author is: Library::Author
```
<br/>

## 클래스 메소드 내에서 `self` 정의
여기서 클래스 인스턴스 메소드와 클래스 메소드 간 차이에 대해 깊이있게 다루지는 않으나 간단하게 정의하자면 다음과 같다.
* 클래스 메소드는 해당 클래스의 개별 인스턴스가 호출할 수 있는 메소드가 아닌 클래스 컨텍스트 내부에서만 참조되는 메소드를 의미한다.
* 클래스 인스턴스 메소드는 클래스 오브젝트 자신은 호출할 수 없지만 해당 클래스의 모든 인스턴스가 호출할 수 있는 메소드를 의미한다.

상기 정의에 기반하여 아래 예제코드에 `name`이라는 클래스 메소드를 `Author` 클래스에 추가했다.(`self` 로 선언한 클래스 내 컨텍스트 자신을 참조한다.) 이 예제에서 필수는 아니지만 `name`이라는 메소드가 `@@name` 클래스 변수에 대한 접근자 프로퍼티로써 보편적으로 사용되는 패턴을 보여주기 위해 추가했다.

```rb
# class-method.rb
class Author
    # Define class variable
    @@name = "John Doe"

    # Getter method
    def self.name
        puts "Self inside class method is: #{self}"
        return @@name
    end
end

puts "Author class method 'name' is: #{Author.name}"
```

여기서 `self.name` 이라는 클래스 메소드 네이밍에서 알아차릴 수 있듯이 클래스 메소드 안에 호출되는 `self`가 부모 클래스를 가리키고 있다는것을 충분히 이해 할 수 있을 것이다. 또한 예제에서 `Author.name` 도 함께 호출하여 클래스 메소드가 어떤식으로 동작하는지 볼 수 있다.
<br/>
<br/>

## 클래스 인스턴스 메소드 내에서 `self` 정의
앞에서 언급했듯이 클래스 인스턴스 메소드는 해당 클래스를 상속하고 있는 모든 인스턴스에서 참조 할 수 있지만, 클래스 오브젝트 자신이 직접 호출 할 수 없는 메소드다. 루비에서 클래스 인스턴스를 정의하려면 단순히 선언할 때 클래스 내에서 `self` 접두사를 제외하면 된다. 이 경우 아래 예제에서 보이듯이 단순히 `def name` 만 하면 동작한다.

```rb
# class-instance-method.rb
class Author

    # Instance method
    def name
        puts "Self inside class instance method is: #{self}"
        puts "Self.class inside class instance method is: #{self.class}"
        return "John Doe"
    end
end

# Define instance
author = Author.new
puts "Author class instance method 'name' is: #{author.name}"
```

해당 메소드는 인스턴스 메소드이기 때문에, `Author` 클래스에서 새로운 인스턴스를 생성 하기 전 까지 해당 메소드를 호출 할 수 없다. 인스턴스를 생성한 이후에 `name` 메소드를 호출 할 수 있고 출력값을 가져 올 수 있다. 클래스 메소드에서 클래스를 바로 참조하던 `self` 와는 다르게 클래스 인스턴스 메소드에서 `self` 는 실행 되는 인스턴스를 참조한다. 이러한 결과 출력 되는 값은 `Author` 클래스의 인스턴스의 메모리 주소값을 출력한다.

```shell
Self inside class instance method is: #<Author:0x00000002bd77c8>
Self.class inside class instance method is: Author
Author class instance method 'name' is: John Doe
```

## 클래스 싱글턴 패턴 내에서의 `self` 정의
마지막으로, 클래스 싱글턴 메소드에서 사용되는 `self` 의 컨텍스트에 대해서 알아보자. 싱글턴 메소드란 단 하나의 인스턴스 오브젝트에서만 사용되며, 다른 인스턴스 오브젝트 또는 부모 클래스 오브젝트 모두 참조가 되지 않는 메소드를 말한다.<br/>
예시를 들어 더 부연설명을 하자면 `Author` 클래스를 선언했으나, `Author` 클래스 선언시, 아무런 메소드도 선언을 하지 않았다고 가정하자. 대신에 `author`라는 새로운 인스턴스를 생성하여 `def [INSTANCE].[METHOD]` 를 생성한다고 생각해보자. 아래 예제에서는 `def author.name` 메소드를 선언 해 보겠다.

```rb
# class-singleton-method.rb
class Author

end

# Define instance
author = Author.new

# Singleton method
def author.name
    puts "Self inside class singleton method is: #{self}"
    puts "Self.class inside class singleton method is: #{self.class}"
    return "John Doe"
end

puts "Author class singleton method 'name' is: #{author.name}"

# Define second instance without singleton method
new_author = Author.new
puts "New class method 'name' should be undefined: #{new_author.name}"
```

위의 예제코드에서 볼 수 있듯이, 싱글턴 메소드 내에서 `self` 를 출력 하면서 `name` 메소드가 단지 `author` 인스턴스에만 선언 되었다는 것을 확인하기 위해서 `new_author` 라는 두번째 클래스를 생성했다. 다시 한번 언급하지만 싱글턴 메소드 컨텍스트를 출력하며 `self` 는 해당 인스턴스의 주소값을 참조한다.

```shell
Self inside class singleton method is: #<Author:0x00000002d36fd8>
Self.class inside class singleton method is: Author
Author class singleton method 'name' is: John Doe
class-singleton-method.rb:20:in `<main>': undefined method `name' for #<Author:0x00000002d36d80> (NoMethodError)
```

Original Source:
[Self in Ruby: A Comprehensive Overview](https://airbrake.io/blog/ruby/self-ruby-overview)