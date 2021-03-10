### *해당 글은 [Ruby on rails 포럼](https://discuss.rubyonrails.org/) 중 일부분을 발췌하여 요약했음을 알려드립니다.*

## Concerns 를 좀 더 쉽게 이해하기

1. `Concerns` 는 무엇인가?<br>
기본적으로 `Concerns` 는 `Ruby`의 `module` 과 비슷한 기능을 한다. 즉, 유지보수 및 각 기능의 맥락에 맞게 분리하는 기능을 담당하는 일종의 믹스인이다.
2. 그럼 `Concerns` 와 `module` 의 차이점은 무엇인가?<br>
차이점이라고 한다면 실행방법, 가독성의 차이점이 있다. 여기 Rails Core 팀인 [`Kasper Timm Hansen`](https://discuss.rubyonrails.org/t/helping-devs-understand-concerns-faster/74619/20)과 [`DHH`](https://discuss.rubyonrails.org/t/helping-devs-understand-concerns-faster/74619/21) 에 따르면 코드상에서 메소드로 호출 할 것인지, Ruby module 처럼 호출 할 것인지 차이점이 있다는 것이다. 여기 아래 예제를 살펴보자.
```rb
# app/models/user.rb
class User < ApplicationRecord
  extend ActiveSupport::Concern

  include Notifiee
end

# app/models/user/notifiee.rb
module User::Notifiee
  # Philosophically a concern, i.e. a role a model can play, but:
  # extend ActiveSupport::Concern is not needed without included/class_methods calls.

  def notifications
    @notifications ||= User::Notifications.new(self)
  end
end

# app/models/user/notifications.rb
class User::Notifications
  attr_reader :user
  delegate :person, to: :user

  def initialize(user)
    @user = user
  end
end
```

위 처럼 `User::Notifiee` 와 `User::Notifications` 를 구성했다고 가정하자. 이 두가지 클래스는 동일한 기능을 하지만 클래스를 호출하는 방법이 조금 다르다. 호출 방식은 다음과 같다.

```rb
current_user.notifications
User::Notifications.new(current_user)
```

차이점이 보이는가? `Concerns`로 한번 상속받은 메소드들은 마치 해당 클래스의 메소드처럼 호출이 가능하다. 하지만 `module`에서 일반적인 기능을 호출하는 방식은 클래스에 인자값들을 넘겨서 새로운 클래스를 만드는 방식이다. 

3. 그럼 둘 중 어느것이 더 나은 것인가?<br>
이것이 포럼에서 쟁점이 된 주요한 논제이다. 이러한 기능이 없어도 개발하는데 전혀 문제가 없고 사용하는 이들이 많지 않기 때문이다. [`DHH`도 언급하였지만](https://discuss.rubyonrails.org/t/helping-devs-understand-concerns-faster/74619/22), 이건 순전히 개발자 본인의 취향이다. 이걸 따르든, 따르지 않든 기능상에 아무런 지장이 없다.

4. 그래도 호출하는 방법이 다른데 사용하는 경우가 다르지 않을까?<br>


Original Source:
[Helping devs understand concerns faster](https://discuss.rubyonrails.org/t/helping-devs-understand-concerns-faster/74619)
