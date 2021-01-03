## Rails 5: 여러 컨텍스트를 함께 검증하기

이 포스팅은 BigBinary 에서 연재하고 있는 [Rails 5 시리즈](https://bigbinary.com/blog/categories/rails-5)의 일부입니다.

`Active Record`의 [검증 기능](https://guides.rubyonrails.org/active_record_validations.html)은 Rails 에서 대표적으로 가장 많이 사용되는 기능이다. 그보다 살짝 덜 알려져 있는 기능이 있는데, 이는 사용자 정의 컨텍스트를 검증하는 기능이다.

해당 긴을 잘만 사용한다면 컨텍스트 검증 코드를 좀 더 깔끔하게 작성 할 수 있다. 컨텍스트 검증을 이해하기 위해 몇가지 단계를 통해 폼 데이터를 전송하는 예제를 살펴보도록 하겠다.

```rb
class MultiStepForm < ActiveRecord::Base
  validate :personal_info
  validate :education, on: :create
  validate :work_experience, on: :update
  validate :final_step, on: :submission

  def personal_info
    # validation logic goes here..
  end

  # Smiliary all the validation methods go here.
end
```

위 예제의 4가지 검증 기능을 하나하나씩 살펴보도록 하자.

1. `personal_info` 검증 기능에는 컨텍스트가 정의되어 있지 않다.(`on:` 키워드가 없다.) 컨텍스트가 없는 검증 기능은 모델의 저장 기능이 호출 될 때마다 실행된다. [여기](https://guides.rubyonrails.org/active_record_validations.html) 에서 모든 트리거들을 한번 살펴보기 바란다.

2. `education` 검증에는 `:create` 컨텍스트가 있다. 이는 오로지 새로운 오브젝트가 생성될때만 동작한다.

3. `work_experienc` 검증은 `:update` 컨텍스트가 있고 업데이트가 될때만 실행된다. `:create` 와 `:update` 두 컨텍스트만 미리 지정되어 있는 상태이다.

4. `final_step`은 사용자정의 컨텍스트인 `:submission` 일때만 실행된다. 위의 경우와는 다르게, 해당 컨텍스트를 다음과 같이 명시적으로 실행시켜줘야 한다.

```rb
form = MultiStepForm.new

# Either
form.valid?(:submission)

# Or
form.save(context: :submission)
```

`valid?` 메소드는 인자로 받은 컨텍스트에 대한 검증을 실행하고 이에 대한 `errors` 클래스 정보를 받는다. `save` 메소드는 인자로 받은 컨텍스트에서 `valid?`를 먼저 호출하고 검증을 통과한 경우에만 저장을 한다. 통과하지 못하면 `errors` 에 데이터를 업데이트한다.

여기서 한가지 유의해야 할 점은 컨텍스트를 명시적으로 검증하는 경우 `:create` 와 `:update` 를 포함한 다른 모든 컨텍스트를 검증하지 않고 통과시킨다.

이제 컨텍스트 검증을 이해했으니, 여러 컨텍스트를 함께 검증하는 `Rails 5` [기능](https://github.com/rails/rails/pull/21535)을 알아보도록 하자.

우선, 처음에 언급했던 예제를 아래와 같이 변경해보자.

```rb
class MultiStepForm < ActiveRecord::Base
  validate :personal_info, on: :personal_submission
  validate :education, on: :education_submission
  validate :work_experience, on: :work_ex_submission
  validate :final_step, on: :final_submission

  def personal_info
    # code goes here..
  end

  # Smiliary all the validation methods go here.
end
```

각 절차에 맞추어 모델 내에 이전 절차들을 검증하고 다음 절차들을 검증하지 않도록 구성해보자. `Rails 5` 이전에는 아래와 같이 작성을 했었다.

```rb
class MultiStepForm < ActiveRecord::Base
  #...

  def save_personal_info
    self.save if self.valid?(:personal_submission)
  end

  def save_education
    self.save if self.valid?(:personal_submission)
              && self.valid?(:education_submission)
  end

  def save_work_experience
    self.save if self.valid?(:personal_submission)
              && self.valid?(:education_submission)
              && self.valid?(:work_ex_submission)
  end

  # And so on...
end
```

위의 예제에서 볼 수 있듯이, `valid?` 메소드는 딱 하나의 컨텍스트를 인자로 받아 실행할 수 있다. 이 때문에 매번 `valid?` 를 반복적으로 실행하고 있는 것을 확인 할 수 있다.

아래 예제에서는 `Rails 5`에서 좀 더 깔끔하게 `valid?` 와 `invalid?`를 배열 형태로 인자를 받아 검증하는 코드를 구성했다.

```rb
class MultiStepForm < ActiveRecord::Base
  #...

  def save_personal_info
    self.save if self.valid?(:personal_submission)
  end

  def save_education
    self.save if self.valid?([:personal_submission,
                              :education_submission])
  end

  def save_work_experience
    self.save if self.valid?([:personal_submission,
                              :education_submission,
                              :work_ex_submission])
  end
end
```

이 코드가 첫번째 검증 코드보다 조금 더 깔끔하다.

Original Source:
[Self in Ruby: A Comprehensive Overview](https://bigbinary.com/blog/validate-multiple-contexts-in-rails-5)
