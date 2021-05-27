## Tip: Automatically cast params with the Rails Attributes API

A common practice in Rails apps is to extract logic into plain-old Ruby objects (POROs). But often you are passing data to these objects directly from controller `params` and the data comes in as strings.

```rb
class SalesReport
  attr_accessor :start_date, :end_date, :min_items

  def initialize(params = {})
    @start_date = params[:start_date]
    @end_date = params[:end_date]
    @min_items = params[:min_items]
  end

  def run!
    # Do some cool stuff
  end
end

report = SalesReport.new(start_date: "2020-01-01", end_date: "2020-03-01", min_items: "10")

# But the data is just stored as strings :(
report.start_date
# => "2020-01-01"
report.min_items
# => "10"
```

You probably want `start_date` to be a date and `min_items` to be an integer. You could add your own basic type casting to the constructor.

```rb
class SalesReport
  attr_accessor :start_date, :end_date, :min_items

  def initialize(params)
    @start_date = Date.parse(params[:start_date])
    @end_date = Date.parse(params[:end_date])
    @min_items = params[:min_items].to_i
  end

  def run!
    # Do some cool stuff
  end
end
```

But even better, you could take advantage of the [Attributes API](https://edgeapi.rubyonrails.org/classes/ActiveRecord/Attributes/ClassMethods.html#method-i-attribute) to handle this casting automatically.

### Usage
The Rails Attributes API is used under-the-hood to type cast attributes for `ActiveRecord` models. When you query for a model that has a `datetime` column in the database and the Ruby object that gets pulled out has a `DateTime` field – that’s the Attributes API at work.

We can spruce up our report model by mixing in the `ActiveModel::Model` and `ActiveModel::Attributes` modules.

```rb
class SalesReport
  include ActiveModel::Model
  include ActiveModel::Attributes

  attribute :start_date, :date
  attribute :end_date, :date
  attribute :min_items, :integer

  def run!
    # Do some cool stuff
  end
end

report = SalesReport.new(start_date: "2020-01-01", end_date: "2020-03-01", min_items: "10")

# Now the attributes are native types!

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
