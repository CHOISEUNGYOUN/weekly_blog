# 루비 메타프로그래밍 : `send` 와 `public_send` 메소드

동적 코드를 작성하는 것은 어려운 작업이다. 다행히도 루비에서는 메타프로그래밍을 지원하고 있기 때문에 코드 작성을 손쉽게 할 수 있다.

메타프로그래밍은 클래스를 작성하거나 메소드를 동적으로 호출할 수 있게 도와주는 강력한 테크닉이기 때문에 반복 작성되는 코드를 줄일 수 있다. 단점으로는 일반적인 코드를 작성하는 것 보다 훨씬 어려운 작업이기 때문에 개인적으로는 원하고자 하는 방향에 맞는 코드 작성이 메타프로그래밍 테크닉으로 작성하는 것 이외에 방법이 없을 때 사용하는 것을 권한다.

이 글에서는 루비의 `send`와 `public_send` 를 다루고자 한다.

우선 이 두 메소드가 어떤 역할을 하는지 살펴보자.

## Send
이 메소드는 문자열이나 심볼을 인자로 받아 객체에 메세지를 전달한다. 두번째 인자는 해당 메소드가 받고자 하는 인자들을 지칭한다.

아래 예시를 살펴보자.

```rb
full_name = "Mia Smith Jr."
full_name.send("count", "i") # 2
full_name.send(:upcase) # "MIA SMITH JR."
```

해당 예시에서 `full_name` 변수에 `Mia Smith Jr.` 문자열을 선언하였다. 이후 `send` 메소드를 활용하여 문자열의 `count`와 `upcase` 메소드를 사용했다. `count` 메소드는 찾고자 하는 문자를 인자로 받기 때문에 두번째 인자로 `"i"` 문자열을 넘겨주었다.

`send`의 동작방식은 첫번째 인자를 넘겨 객체에 존재하는 메소드를 실행한다. 해당 객체가 응답한다면 원하는 결과값을 얻게 된다. 그렇지 않은 경우에는 에러를 반환하게 된다.

응답이 잘 되었는지 확인을 하고 싶다면 `respond_to?` 메소드를 사용하여 확인 할 수 있다. 해당 객체가 원하는 메소드를 잘 실행했는지 미리 확인 할 수 있는 좋은 방법이다.

예를 들어 `split` 메소드가 잘 실행되었는지 확인하고 싶다면 아래와 같이 작성하면 된다.

```rb
full_name.respond_to?(:split) # true
```

앞서 언급하였듯이 문자열이나 심볼을 `send`와 `public_send` 메소드에 넘겨주어 실행 할 수 있다. 가능하다면 문자열보다는 효율적으로 설계된 심볼을 인자로 넘겨주는 것이 좋다. 

## Public Send
이 메소드는 `send`와 완벽히 동일하게 동작한다. 여기서 조금 다른 점은 공개된 메소드들만 호출한다는 점이다. 비공개(private) 메소드나 보호된(protected) 들은 호출되지 않는다.

아래 `public_send`의 예시를 살펴보자.

```rb
class DeviceReporter
  def initialize(device)
    @device = device
  end

  # use public_send method to send a message to the device object.
  # public_send를 사용하여 device 객체에 메세지를 보낸다.
  def call
    # logic to generate a report
    p device.public_send(:tech_details)
    # more logic
  end

  private

  attr_reader :device
end

class Device
  def initialize(name, type, price, model)
    @name = name
    @type = type
    @price = price
    @model = model
  end

  def tech_details
    {
      "name": name,
      "type": type,
      "price": price,
      "model": model
    }
  end

  private
  attr_reader :name, :type, :price, :model

  def total_price
    price.to_i + (price.to_i * 0.16)
  end
end

device = Device.new("Moto G8 Plus", "cellphone", "350", "G8")
deviceReporter = DeviceReporter.new(device).call
```

이 예시에서 볼 수 있듯이 `Device` 클래스와 `DeviceReporter` 두 클래스가 있다. 여기서 새로운 `Device` 인스턴스를 생성하고 해당 인스턴스를 `DeviceReporter` 클래스의 인자로 넘긴다. 그러면 `Reporter`클래스는 `public_send` 메소드를 사용하여 메세지를 `Device` 객체에 넘겨주게 되는데 여기 예시에서는 `tech_details` 메소드를 사용한다. `Device` 메소드가 이 메세지에 응답하면 에러를 반환하지 않는 대신 `tech_details` 메소드의 결과값인 해시를 반환하게 된다.

여기서 살펴볼 수 있듯이 `Device`의 공개 메소드를 호출하는 것을 확인 할 수 있다. `public_send`를 사용해서 비공개 메소드나 보호된 메소드(여기서는 `total_price` 메소드)를 호출하게 되면 에러를 반환하게 된다. 이는 `public_send` 메소드가 다른 객체의 내부적인 세부사항들을 접근하지 못하게 하기 때문이다 (예: 해당 메소드가 외부에서 사용되지 않도록 보장).

이는 객체들이 세부사항들을 비공개로 유지함과 동시에 외부 객체에 공개되지 않아야 할 내용들을 캡슐화 하는데 유용하다.

## 이런 메소드를 왜 사용해야 할까?
`send`와 `public_send`를 사용함으로써 얻는 이점들은 어떤 것들이 있을까? 답은 간단하다. 메세지를 동적으로 보낼 수 있기 때문이다.

`send` 메소드를 사용하는 다른 예제를 살펴보자.

```rb
class BookService
  def initialize(params)
    @params = params
  end

  def call
    send("filter_by_#{params[:filter]}")
  end

  private
  
  attr_reader :params

  def filter_by_author
    p "filter by author"
  end

  def filter_by_editorial
    p "filter by editorial"
  end
end


books = BookService.new({"filter": "author"}).call
```

위 예시에서는 `BookService` 에서 특정 매개변수를 컨스트럭터에서 받는 공개 메소드를 구현했다. 이 경우 `public_send`와 달리 `send`를 사용하는데 이는 `self`에 메세지를 전달하고 싶기 때문에다. 즉 `self` 내에서 비공개 메소드나 보호된 메소드를 호출하려고 하기 때문에 `public_send`를 사용할 필요가 없게 된다.

해당 `call` 메소드는 동적으로 메세지를 `self`로 부터 받게된다. 이 방식을 사용하면 `if`문을 여러번 사용하는 것 보다 훨씬 깔끔하게 작성 할 수 있다.

이번에는 `public_send`의 다른 예시를 살펴보고자 한다.

```rb
class NotifierService
  USER_FIELDS = [:name, :age]

  def initialize(notifier_type, message, user)
    @notifier_type = notifier_type
    @message = message
    @user = user
  end

  def notify
    @fields = user_fields
    send("notify_by_#{notifier_type}")
  end

  private

  attr_accessor :notifier_type, :user, :message

  def notify_by_email; end

  def notify_by_sms; end

  def notify_by_whatsapp; end

  def user_fields
    USER_FIELDS.each_with_object({}) do |field, memo|
      memo["#{field}"] = user.public_send(field) 
    end
  end
end


class User
  def initialize(name, last_name, age)
    @name = name
    @age = age
    @last_name = last_name
  end

  def get_full_name
    name + " " + last_name
  end

  attr_accessor :name, :last_name

  private

  attr_reader :age

  def greet
    "Hi I am #{name} and I am #{age}"
  end
end

u = User.new("carl", "smith", 29)
ns = NotifierService.new("sms", "Hello there!!", u).notify
```

이 예시에서 `notifier_type`, `message`와 `user`를 인자로 가지고 있는 `NotifierService` 클래스를 볼 수 있다. 여기서 `user_fields` 비공개 메소드를 호출하고 그 결과값을 `fields` 전역변수에 저장하는 `notify` 라는 공개 메소드가 있는 것을 확인 할 수 있다.

`user_fields` 메소드는 사용자 속성을 순회하여 객체에 저장하는 것을 확인 할 수 있다. 각 속성을 저장하기 위해 `public_send`를 사용하여 동적으로 메세지를 보내 사용자 객체에 값을 채워넣게 된다,

이후 `notify` 메소드에서는 `send`를 사용하여 `notifier_type` 인자에 따라 실행되는 것을 확인 할 수 있다.

여기까지 이해한 것을 바탕으로 `User` 클래스를 살펴보자.

`User` 클래스는 `name`, `last_name`, `age` 를 컨스트럭터에서 저장하고 있다. 또한 공개 및 비공개 메소드가 함께 존재한다. 이 클래스의 중요한 부분은 오직 `name`과 `last_name` 에만 접근을 허용하고 있다는 점이다. `age`는 비공개 속성이기 때문에 오직 클래스에서만 접근이 가능하다.

이 코드를 실행하게 되면 에러를 발생시키는데 이는 `NotifierService`에 선언된 상수에 `user` 객체의 비공개 메소드가 있기 때문이다. 결국 해당 배열을 순회하여 `public_send`를 호출하면 비공개 메소드를 호출하려고 한다는 에러메세지와 함께 실행이 중단된다.

이는 `public_method`를 사용하는 주된 이유이다. `user` 객체에 비공개로 정의된 특정 요소나 메소드를 호출하는 경우 `public_send`는 다른 객체의 내부 사항을 접근하려고 한다는 점을 알려주게 된다.

해당 에러를 해결하기 위해선 `respond_to?`를 사용하여 아래와 같이 해결할 수 있다.

```rb
def user_fields
  USER_FIELDS.each_with_object({}) do |field, memo|
    memo["#{field}"] = user.public_send(field) if user.respond_to?(field)
  end
end
```

위 조건으로 인해 `user_field`에 비공개 메소드나 요소를 추가하더라도 에러를 반환하지 않게 된다.

## 요약
* `send`를 사용하여 같은 클래스(self)의 메소드나 인자를 동적으로 호출 할 수 있다.
* `public_send`를 사용하여 다른 객체의 메소드나 인자를 동적으로 호출 할 수 있다. 이 경우 비공개 메소드나 인자를 호출하지 않도록 보장해준다.
* `respond_to?`를 사용하여 다른 객체의 메소드나 인자를 호출하기 전 호출 가능한지 여부를 확인 할 수 있다.

Original Source:
[Metaprogramming With Ruby: Send and Public Send Methods](https://betterprogramming.pub/metaprogramming-with-ruby-send-and-public-send-methods-f85aa8f91366)
