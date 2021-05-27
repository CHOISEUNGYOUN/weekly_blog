## Ralis Attributes API를 활용한 params 자동 캐스팅.

Rails 어플리케이션을 구축하면서 가장 일반적으로 사용하는 패턴은 로직에 담긴 데이터를 plain-old Ruby 객체(POROs)에 담는 일이다. 하지만 종종 컨트롤러에서 `params`에 담겨있는 문자열 데이터를 받아 직접 처리하는 경우도 있다.

```rb
class SalesReport
  attr_accessor :start_date, :end_date, :min_items

  def initialize(params = {})
    @start_date = params[:start_date]
    @end_date = params[:end_date]
    @min_items = params[:min_items]
  end

  def run!
    # 실행 메소드 구현
  end
end

report = SalesReport.new(start_date: "2020-01-01", end_date: "2020-03-01", min_items: "10")

# 아래 데이터는 모두 문자열임.
report.start_date
# => "2020-01-01"
report.min_items
# => "10"
```

당신은 위 예제에 있는 `start_date`는 date 속성으로, `min_items`는 integer 속성으로 할당되길 원할 것이다. 이런 경우 constructor에 간단한 타입 캐스팅을 추가하면 해당 기능을 구현할 수 있다.

```rb
class SalesReport
  attr_accessor :start_date, :end_date, :min_items

  def initialize(params)
    @start_date = Date.parse(params[:start_date])
    @end_date = Date.parse(params[:end_date])
    @min_items = params[:min_items].to_i
  end

  def run!
    # 실행 메소드 구현
  end
end
```

위 방법도 좋지만, Rails 의 [Attributes API](https://edgeapi.rubyonrails.org/classes/ActiveRecord/Attributes/ClassMethods.html#method-i-attribute) 를 사용하여 이러한 타입 캐스팅을 자동으로 구현 한다면 더욱 더 깔끔하고 편리한 코드를 작성할 수 있다.

### 사용법
Rails의 Attributes API는 `ActiveRecord` 모델의 타입을 자동으로 캐스팅하기 위해 사용되고 있다.
예를 들어 `datetime` 컬럼을 가지고 있는 모델에 쿼리를 요청하면 `Datetime` 필드 속성을 가진 Ruby 객체를 호출 할 때 Attributes API가 타입 캐스팅을 해준다. 우리가 모르는 사이에 DB와 Ruby 간의 데이터를 자동으로 맞춰주는 것이다.

`ActiveModel::Model` 과 `ActiveModel::Attributes` 모듈을 사용하면 아래 예제의 `SalesReport` 모델을 깔끔하게 타입 캐스팅 처리 할 수 있다.

```rb
class SalesReport
  include ActiveModel::Model
  include ActiveModel::Attributes

  attribute :start_date, :date
  attribute :end_date, :date
  attribute :min_items, :integer

  def run!
    # 실행 메소드 구현
  end
end

report = SalesReport.new(start_date: "2020-01-01", end_date: "2020-03-01", min_items: "10")

# 아래 객체의 데이터는 이제 ruby native 타입으로 캐스팅 되었음.

report.start_date
# => Wed, 01 Jan 2020
report.min_items
# => 10
```

This pattern is great for reducing boilerplate code in form objects, report objects, or any other Model-ish Ruby class in your Rails apps. Let the framework do the type casting for you, instead of trying to reimplement it yourself!

### Options
> As of Rails 6.1, this module is technically a private API. Use at your own risk!

The Attribute API will automatically handle type casting for most primitives. All of the basics are covered.

```rb
attribute :start_date, :date
attribute :max_size, :integer
attribute :enabled, :boolean
attribute :score, :float
```

You can find the full list of out-of-the-box types here: [activemodel/lib/active_model/type](https://github.com/rails/rails/tree/v6.0.2.1/activemodel/lib/active_model/type).

The coolest part is that the types are very robust in what kind of input they accept. For example, the boolean Attribute type works with any of these values for `false`:

```rb
FALSE_VALUES = [
  false, 0,
  "0", :"0",
  "f", :f,
  "F", :F,
  "false", :false,
  "FALSE", :FALSE,
  "off", :off,
  "OFF", :OFF,
]
```

You can also register your own custom types that implement `cast` and `serialize`:

```rb
ActiveRecord::Type.register(:zip_code, ZipCodeType)

class ZipCodeType < ActiveRecord::Type::Value
  def cast(value)
    ZipCode.new(value) # cast to your own ZipCode class for special handling
  end

  def serialize(value)
    value.to_s
  end
end
```

Additionally, you can set a default value for with the Attributes API:

```rb
attribute :start_date, :date, default: 30.days.ago
attribute :max_size, :integer, default: 15
attribute :enabled, :boolean, default: true
attribute :score, :float, default: 9.75
```


추가 참고 사항:

Rails API: [Helpers](https://edgeapi.rubyonrails.org/classes/ActiveRecord/Attributes/ClassMethods.html#method-i-attribute)

[Rails’ hidden type system](https://blog.dario-hamidi.de/a/rails-hidden-type-system)

Original Source:
[Tip: Building lightweight components with Rails Helpers and Stimulus](https://boringrails.com/tips/lightweight-components-with-helpers-stimulus)
