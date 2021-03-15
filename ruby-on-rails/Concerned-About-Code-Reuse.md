## Concerned about Code Reuse?

Right out of the gate, Ruby gives us some powerful ways to re-use instance and class methods without relying on inheritance. Modules in Ruby can be used to mixin methods to classes fairly easily. For example, we can add new instance methods using include.

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

Now we’re able to call any methods defined in our `DogFort` Module as if they were simply slipped into (included) into our `Dog` class.
```rb
dog_instance = Dog.new
dog_instance.call_dog
# => "this is dog!"
```
Using Modules a fairly easy way to re-use methods, if you want you can `extend` a Module to add methods to a class directly.

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
Now if we were to call `Dog.new.board_the_doors` we would get an error, since we’ve added it as a class method instead.

```rb
Dog.board_the_doors
# => "no catz allowed"

Dog.class
# => Class
```
Sweet! Though what if you wanted to add an instance method and a class method to a class. We could have two Modules, one to be included and one to be extended, wouldn’t be to hard but it would be nice if we only had to use one include statement, especially if the two Modules are related. So is it possible to add instance and class methods with only one include statement? Of course…

### Enter Concerns
A concern is a Module that adds instance methods (like `Dog.new.call_dog`) and class methods (like `Dog.board_the_doors`) to a class. If you’ve poked around the Rails source code you’ll see this everywhere. It’s so common that Active Support added a helper Module to create concerns. To use it require ActiveSupport and then `extend ActiveSupport::Concern`

```rb
require 'active_support/concern'

module DogFort
  extend ActiveSupport::Concern
  # ...
end
```
Now any methods you put into this Module will be instance methods (methods on a new instance of a class `Dog.new`) and any methods that you put into a Module named `ClassMethods` will be added on to the class directly (such as `Dog`).

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
So now when we add this new Module to a class, we’ll get instance and class methods

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
Pretty cool huh?

### Included
That’s not all, Active Support also gives us a special method called included that we can use to call methods during include time. If you add `included` to your `ActiveSupport::Concern` any code in there will be called when it is included

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

So when we include `DogCatcher` in a class it’s included block will be called immediately.

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
While this is a contrived example, you can imagine wanting to maybe make a concern for Rails controllers and wanting to add `before_filter`'s to our code. We can do this easily adding the included block.

### Is this magic?
Nope, under the hood we’re just using good old fashioned Ruby. If you want to learn more about all the fun things you can do with Modules I recommend checking out one of my favorite Ruby books Metaprogramming Ruby and Dave Thomas also has a fantastic screencast series.

### Gotcha
When you’re writing Modules I guarantee that you’ll slip up and accidentally try to create a class method using `self` or `class << self` but it won’t work because it’s now a method on the Module.

```rb
module DogFort
  def self.call_dog
    puts "this is dog!"
  end
end
```
In the example above the context of `self` is actually the Module object `DogFort` so when we include it into another class we won’t see the method.

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
If you want to use that method in this context you will need to call the Module directly

```rb
DogFort.call_dog
# => "this is dog!"
puts DogFort.class
# => Module
```

Original Source:
[Concerned about Code Reuse?](https://schneems.com/post/21380060358/concerned-about-code-reuse)
