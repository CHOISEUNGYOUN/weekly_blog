## 코드 재사용성을 고민하고(Concerned) 있는가?

코드 재사용성을 고민하고 있는 당신을 위해 Ruby는 클래스 인스턴스와 메소드를 상속없이 재사용 가능한 방법을 제시해준다. Ruby 언어에 존재하는 Module은 클래스 믹스인으로 쉽게 사용이 가능하다. 예를 들어 아래 예제와 같이 Module을 `include` 하여 사용할 수 있다.

```rb
module DogFort
  def call_dog
    puts "this is dog!"
  end
end

class Dog
  include DogFort
end
```

이와 같이 작성하면 `DogFort` Module에서 구현한 모든 메소드들을 `Dog` 클래스에서 사용 가능하다.

```rb
dog_instance = Dog.new
dog_instance.call_dog
# => "this is dog!"
```

또한 `extend` 를 선언하면 Module 메소드들을 클래스 메소드로 손쉽게 사용 할 수 있다..

```rb
module DogFort
  def board_the_doors
    puts "no catz allowed"
  end
end

class Dog
  extend DogFort
end
```

하지만 `Dog.new.board_the_doors` 형태 처럼 인스턴스 메소드로 부르려고 한다면 에러가 날 것이다. 이는 `extend` 로 포함한 Module이 클래스 메소드이기 때문이다.

```rb
Dog.board_the_doors
# => "no catz allowed"

Dog.class
# => Class
```

멋지지 않는가? 하지만 클래스 메소드와 인스턴스 메소드 둘다 추가하려면 어떻게 해야할까? 아마 `include` 와 `extend` 로 호출되는 두개의 Module이 필요할 것이다. 그리 어려운 것은 아니나 두 Module 모두 연관성이 있다면 하나의 Module로 구성해서 호출하는 것이 더 나을 것이다. 클래스 메소드와 인스턴스 메소드 둘다 하나의 Module에 포함해서 선언하는 방법이 없을까? 당연히 있다!

### Concerns 입문

Concern은 Module의 인스턴스 메소드(예: `Dog.new.call_dog`) 와 클래스 메소드 (예: `Dog.board_the_doors`)를 하나로 합친 Module이다. Rails 소스코드들을 조금만 훑어본다면 여기저기에서 사용되고 있는 것을 발견 할 수 있다. `ActiveSupport` 클래스에 Concern을 만들기 위해 helper Module을 추가하는것은 흔한 패턴이다. 이를 사용하기 위해선 `ActiveSupport`를 가져오고 `extend ActiveSupport::Concern`을 하면 된다.

```rb
require 'active_support/concern'

module DogFort
  extend ActiveSupport::Concern
  # ...
end
```

이제 Module에 추가하는 모든 메소드들은 인스턴스 메소드가 된다.(여기 예시에서는 `Dog` 클래스의 인스턴스 메소드.) 그리고 `ClassMethods` Module 하위에 존재하는 모든 메소드들은 클래스 메소드가 된다. (여기 예시에서는 `Dog` 클래스에서 바로 호출되는 메소드.)

```rb
require 'active_support/concern'

module YoDawgFort
  extend ActiveSupport::Concern

  def call_dawg
    puts "yo dawg, this is dawg!"
  end


  # Anything in ClassMethods becomes a class method
  module ClassMethods
    def board_the_doors
      puts "yo dawg, no catz allowed"
    end
  end
end
```

이제 이 Module을 클래스에 추가하면 클래스 메소드와 인스턴스 메소드 모두 추가 할 수 있다.

```rb
class YoDawg
  include YoDawgFort
end

YoDawg.board_the_doars
# => "yo dawg, no catz allowed"

yodawg_instance = YoDawg.new
yodawg_instance.call_dawg
# => "yo dawg, this is dawg!"
```
꽤 근사하지 않는가?

### Included

이것 뿐만 아니라 `ActiveSupport` 클래스 에서는 `included` 라는 특수한 기능을 사용할 수 있다. 이 기능은 메소드를 include 하는 과정에서 호출 된다. 만약 `ActiveSupport::Concern` 에 `included` 블럭으로 감싼 코드를 추가한다면 해당 코드들은 Module이 include 될때 호출된다.

```rb
module DogCatcher
  extend ActiveSupport::Concern

  included do
    if self.is_a? Dog
      puts "gotcha!!"
    else
      puts "you may go"
    end
  end
end
```

위 예제의 `DogCatcher` 를 아래 예시와 같이 include 한다면 include 되는 즉시 호출된다.

```rb
class Dog
  include DogCatcher
end
# => "gotcha!!"

class Cat
  include DogCatcher
end
# => "you may go"
```

이 코드는 컨셉에 불과하지만 Rails 컨트롤러에 concern 을 만들어 `before_filter` 를 위한 코드를 구현하는 것을 생각 해볼 수 있다. 이런 것들을 `included` 블럭을 통해서 쉽게 구현 할 수 있다.


### 이 모든 것들이 과연 마법일까?

전혀 그렇지 않다. concern의 이면에는 단지 우리가 오랫동안 사용해온 Ruby 코드들이 존재한다. Module 에 대한 디테일하고 재밌는 요소들을 더 배워보고 싶다면 필자는 [Metaprogramming Ruby 2](https://books.google.co.kr/books/about/Metaprogramming_Ruby_2.html?id=V0iToAEACAAJ&source=kp_cover&redir_esc=y) 책과 [Dave Thomas](https://ruby-doc.org/docs/ruby-doc-bundle/ProgrammingRuby/index.html) 가 저술한 시리즈를 필독하기를 추천한다.

### Module 완전히 이해하기

필자가 확신하건데, 종종 Module을 사용하다 보면 `self` 나 `class << self` 를 사용하여 클래스 메소드를 구현하는 실수를 범한다. 당연히 이것은 Module의 메소드이기 때문에 동작하지 않는다.

```rb
module DogFort
  def self.call_dog
    puts "this is dog!"
  end
end
```

위 예시에서의 `self` 는 Module 오브젝트인 `DogFrot`를 가리키기 때문에 다른 클래스에서 이 모듈을 include 하는 경우 제대로 동작하지 않는다.

```rb
class Wolf
  include DogFort
end

Wolf.call_dog
# NameError: undefined local variable or method `call_dog'

wolf_instance = Wolf.new
wolf_instance.call_dog
# NameError: undefined local variable or method `call_dog'
```

`call_dog` 메소드를 사용하고 싶다면 `DogFort` 모듈을 통해서만 호출이 가능하다.

```rb
DogFort.call_dog
# => "this is dog!"
puts DogFort.class
# => Module
```

Original Source:
[Concerned about Code Reuse?](https://schneems.com/post/21380060358/concerned-about-code-reuse)
