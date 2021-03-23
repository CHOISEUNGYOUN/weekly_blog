### *해당 글은 [Ruby on rails 포럼](https://discuss.rubyonrails.org/) 중 일부분을 발췌하여 요약했음을 알려드립니다.*

## Concerns 를 좀 더 쉽게 이해하기

1. `Concerns` 는 무엇인가?<br>
   기본적으로 `Concerns` 는 `Ruby`의 `module` 과 비슷한 기능을 한다. 즉, 유지보수 및 각 기능의 맥락에 맞게 분리하는 기능을 담당하는 일종의 믹스인이다.
2. 그럼 `Concerns` 와 `module` 의 차이점은 무엇인가?<br>
   차이점이라고 한다면 메소드 관리 측면 및 가독성에서 차이가 있다. 여기 Rails Core 팀인 [`Kasper Timm Hansen`](https://discuss.rubyonrails.org/t/helping-devs-understand-concerns-faster/74619/20)과 [`DHH`](https://discuss.rubyonrails.org/t/helping-devs-understand-concerns-faster/74619/21)가 예시로 언급한 코드들을 하나하나씩 살펴보자.

3. `Module` 활용 예시<br>
   이 예시들은 실제 Basecamp3, Hey 앱에서 사용중인 코드들이다. 이 코드들의 특징은 `ActiveSupport::Concern`을 사용하지 않고 `POROs(Plain Old Ruby Objects)` 룰을 따랐다는 것이다.
  `예시1`

   ```rb
   # app/models/user.rb
   class User < ApplicationRecord
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
   위와 같이 `Concern`을 사용하지 않고도 믹스인을 사용 할 수 있다. 아래와 같이 메소드로 활용 가능하다.

   ```rb
   user = User.new
   user.notifications
   # 안티패턴이지만 아래와 같이 호출도 가능함
   User::Notifications.new(user)
   ```

   `예시2`
   ```rb
   class Identity < ApplicationRecord
     include Accessor, Authenticable, Boxes, ClearancePreapproval, Clipper, Creator, Eventable,
       Examiner, Filer, Member, PasswordReset, Poster, Reaching, Regional, TopicMerger
   end
   module Identity::Accessor
     def accessible_topics
       Topic.joins(:accesses).where(accesses: { contact: contacts })
     end
     def accessible_entries
       Entry.joins(:topic).merge(accessible_topics)
     end
   end
   ```
   위 경우에는 `Identity::Accessor` 모듈이 `Topic`과 `Entry` 테이블에 저장된 데이터들을 `Identity` 인스턴스에서 사용할 수 있게하는 인스턴스 메소드가 된다. 여기서 해당 메소드는 `Identity` 모델의 scope이 되어야한다고 이야기 할 수도 있으나, `Identity` 모델이 다양한 모듈들을 믹스인으로 상속받는다는 것을 감안하면, 용도에 따라 구분을 해놓은 것이다. 설계적 관점에서는 이것이 더 깔끔해 보인다.

4. `Concern` 활용 예시<br>
   아래의 예시를 한번 살펴보자.
   ```rb
   module Eventable
     extend ActiveSupport::Concern

     included do
       has_many :events, as: :eventable, dependent: :destroy
     ...

   class Access < ApplicationRecord
      include Eventable

    class Clip < ApplicationRecord
      include Eventable
   ```

   위 예시의 `Eventable`은 `app/model/concerns`에 위치하고 있다. 폴더 위치로 짐작 할 수 있듯이 해당 모듈은 모든 모델에서 필요한 경우 믹스인으로 상속이 가능하다. 즉, `events` 가 존재하는 모든 모델에서 공용으로 사용할 수 있다는 것이다. 이렇게 사용하는 경우 재활용성이 뛰어나고 Rails에서 추구하는 `DRY`의 원칙을 온전히 따를 수 있다.

5. 그럼 둘 중 어느것이 더 나은 것인가?<br>
   이것이 포럼에서 쟁점이 된 주요한 논제이다. 이러한 기능이 없어도 개발하는데 전혀 문제가 없고 사용하는 이들이 많지 않기 때문이다. [`DHH`도 언급하였지만](https://discuss.rubyonrails.org/t/helping-devs-understand-concerns-faster/74619/22), 이건 순전히 개발자 본인의 취향이다. 이걸 따르든, 따르지 않든 기능상에 아무런 지장이 없다.


Original Source:
[Helping devs understand concerns faster](https://discuss.rubyonrails.org/t/helping-devs-understand-concerns-faster/74619)
