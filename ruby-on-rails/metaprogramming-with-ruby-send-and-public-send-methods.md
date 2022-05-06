# 루비 메타프로그래밍 : `send` 와 `public_send` 메소드

동적 코드를 작성하는 것은 어려운 작업이다. 다행히도 루비에서는 메타프로그래밍을 지원하고 있기 때문에 코드 작성을 손쉽게 할 수 있다.

메타프로그래밍은 클래스를 작성하거나 메소드를 동적으로 호출할 수 있게 도와주는 강력한 테크닉이기 때문에 반복 작성되는 코드를 줄일 수 있다. 단점으로는 일반적인 코드를 작성하는 것 보다 훨씬 어려운 작업이기 때문에 개인적으로는 원하고자 하는 방향에 맞는 코드 작성이 메타프로그래밍 테크닉으로 작성하는 것 이외에 방법이 없을 때 사용하는 것을 권한다.

이 글에서는 루비의 `send`와 `public_send` 를 다루고자 한다.

우선 이 두 메소드가 어떤 역할을 하는지 살펴보자.

## Send
이 메소드는 문자열이나 심볼을 인자로 받아 오브젝트에 메세지를 전달한다. 두번째 인자는 해당 메소드가 받고자 하는 인자들을 지칭한다.

아래 예시를 살펴보자.

```rb
full_name = "Mia Smith Jr."
full_name.send("count", "i") # 2
full_name.send(:upcase) # "MIA SMITH JR."
```

해당 예시에서 `full_name` 변수에 `Mia Smith Jr.` 문자열을 선언하였다. 이후 `send` 메소드를 활용하여 문자열의 `count`와 `upcase` 메소드를 사용했다. `count` 메소드는 찾고자 하는 문자를 인자로 받기 때문에 두번째 인자로 `"i"` 문자열을 넘겨주었다.

`send`의 동작방식은 첫번째 인자를 넘겨 오브젝트에 존재하는 메소드를 실행한다. 해당 오브젝트가 응답한다면 원하는 결과값을 얻게 된다. 그렇지 않은 경우에는 에러를 반환하게 된다.

응답이 잘 되었는지 확인을 하고 싶다면 `respond_to?` 메소드를 사용하여 확인 할 수 있다. 해당 오브젝트가 원하는 메소드를 잘 실행했는지 미리 확인 할 수 있는 좋은 방법이다.

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
  # public_send를 사용하여 device 오브젝트에 메세지를 보낸다.
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

이 예시에서 볼 수 있듯이 `Device` 클래스와 `DeviceReporter` 두 클래스가 있다. 여기서 새로운 `Device` 인스턴스를 생성하고 해당 인스턴스를 `DeviceReporter` 클래스의 인자로 넘긴다. 그러면 `Reporter`클래스는 `public_send` 메소드를 사용하여 메세지를 `Device` 오브젝트에 넘겨주게 되는데 여기 예시에서는 `tech_details` 메소드를 사용한다. `Device` 메소드가 이 메세지에 응답하면 에러를 반환하지 않는 대신 `tech_details` 메소드의 결과값인 해시를 반환하게 된다.

여기서 살펴볼 수 있듯이 `Device`의 공개 메소드를 호출하는 것을 확인 할 수 있다. `public_send`를 사용해서 비공개 메소드나 보호된 메소드(여기서는 `total_price` 메소드)를 호출하게 되면 에러를 반환하게 된다. 이는 `public_send` 메소드가 다른 오브젝트의 내부적인 세부사항들을 접근하지 못하게 하기 때문이다 (예: 해당 메소드가 외부에서 사용되지 않도록 보장).

이는 오브젝트들이 세부사항들을 비공개로 유지함과 동시에 외부 오브젝트에 공개되지 않아야 할 내용들을 캡슐화 하는데 유용하다.

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

In the example above, we have a `book service` class that receives some params in the constructor and has one public method. In this case, we are using the `send` method as opposed to `public_send` because we want to send a message to self. This means we want to execute a method that is part of the same class, so we don’t have to use the `public_send` method since we can call any private or protected methods of `self`.

In the call method, we are dynamically sending a message to `self` depending on the filter that we receive in the constructor. This way, we can avoid having an `if` statement and the code looks quite clean.

Now, let’s see another example using `public_send`:

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

In this example, we have a `notifier service` class that receives a `notifier_type`, `message`, and `user` in the constructor as arguments. Then it has a `notify` public method that first calls the `user_fields` private method and assigns the result to the `fields` global variable.

In the `user_fields` method, we are iterating over a constant that contains a set of user properties. For each property, we use the `public_send` method to send that message to the user object dynamically and assign that property to the hash. That will produce a hash of user properties.

Then the next line inside the `notify` method uses the `send` again to execute the right method based on the `notifier_type` argument that we received.

Until this point, everything looks good. Let’s take a look at the `User` class now.

The user class takes `name`, `last_name`, and `age` in the constructor. It has a public and a private method. The important part of this class is that it has accessors only for `name` and `last_name` — not for `age`. `Age` is a private reader property, which means it should only be accessible to the class and not exposed to other objects.

If we run this code as it is, it will produce an error because the constant that is defined inside the `notifier service` contains a private property from the `user` object. So, when iterating over this array and trying to use the `public_send` method, it will throw an error saying that this is a private method and therefore we can’t call it.

This is where I find the `public_send` method really helpful. If you forget that the `user` object has certain properties or methods defined as private for some reason, this method will tell you that you are trying to violate the internal details of another object.

Now, to fix this error, we can use the `respond_to?` method that I mentioned earlier:

```rb
def user_fields
  USER_FIELDS.each_with_object({}) do |field, memo|
    memo["#{field}"] = user.public_send(field) if user.respond_to?(field)
  end
end
```

With this change, even if we add private methods and properties to the `user fields` constant, it will not throw an error because we are checking if the user responds to that message before trying to use the `public_send` method on it.

## Key Takeaways
* Use the `send` method when you want to call methods and properties dynamically in the same class (`self`).
* Use the `public_send` method when you want to call methods and properties dynamically on other objects. This ensures that you are not calling any private methods or properties.
* Use the `respond_to?` method when you want to know if an object responds to a specific message before actually sending it.

Thanks for reading! I hope you have found this useful.

Original Source:
[Metaprogramming With Ruby: Send and Public Send Methods](https://betterprogramming.pub/metaprogramming-with-ruby-send-and-public-send-methods-f85aa8f91366)
