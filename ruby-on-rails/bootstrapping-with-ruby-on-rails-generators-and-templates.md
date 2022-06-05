# Ruby on Rails 제네레이터와 템플릿을 부트스트래핑하기

레일즈의 [배터리를 포함하는 접근방식(batteries-included, 작업에 필요한 도구들을 가능한 모두 제공하는 방식)](https://en.wikipedia.org/wiki/Batteries_Included)은 가장 큰 장점중에 하나이다. 그 중에서도 레일즈의 제네레이터 덕분에 다른 어떠한 프레임워크도 별 다른 노력없이 처음부터 어플리케이션을 작성할 수 있다.

레일즈를 얼마나 사용했던 간에 필연적으로 제네레이터를 사용해보았을 것이다. 새로운 어플리케이션이 필요하다면 `rails new` 커멘드를 사용한다. 수많은 모델과 뷰를 다루는 스캐폴드를 작성해야한다면 `rails generate scaffold`를 사용하면 된다. 이외에도 다른 여러 제네레이터를 사용하여 빠르게 앱을 작성하고 유연하게 작업을 할 수 있다.

하지만 기본적으로 제공해주는 제네레이터가 모든 문제를 해결해주진 않는다. 아마 제네레이터를 수정하거나 새로운 제네레이터를 만들어야 할 필요가 있을 것이다. 이번 글에서 제네레이터를 어떻게 만들어내는지 살펴보고자 한다. 특히 레일즈에서 템플릿을 활용하여 커스텀 제네레이터를 만드는 것을 중점적으로 다루고자 한다.

## 레일즈 제네레이터란 무엇인가?
Not to be confused with generator functions (which you might be familiar with from Python or Javascript), Rails generators are custom [Thor](https://github.com/rails/thor) commands that focus on, well, generating things.

There are lots of examples. You'll likely be familiar with the model generator (`rails generate model`) for creating new ActiveRecord models or the migration generator (`rails generate migration`) for generating new migrations. There is also `rails generate generator` which — you guessed it — creates a new generator!

Generators can call each other — for example, `rails scaffold` will call numerous other generators — and provide methods to create or modify files, install gems, run specific rake tasks, and much more. Let's create a simple model spec generator to understand how this works.

## Creating Your Own Generator in Ruby on Rails
Run the following:

```zsh
rails generate generator model_spec
```

This will create several new files in `/lib/generators/model_spec.` We can modify `model_spec_generator.rb` in folder `lib/generators/model_spec/` to create a model spec file in the correct directory:

```rb
class ModelSpecGenerator < Rails::Generators::NamedBase
  source_root File.expand_path('templates', __dir__)

  def create_model_spec
    template_file = File.join('spec/models', class_path, "#{file_name}_spec.rb")
    template 'model_spec.rb.erb', template_file
  end
end
```
The `template` command will look for a template file in the `lib/generators/model_spec/templates` directory and render it to the specified location — the `spec/models` directory. The command will replace ERB-style variables found in the template file.

By setting the `source_root`, we let our generator know where it can find referenced template files. Template `model_spec.rb` in folder `lib/generators/model_spec/templates/` could look like this:

```rb
require 'rails_helper'

RSpec.describe <%= class_name %>, :model
  pending "add some examples to (or delete) #{__FILE__}"
end
```
Once you have created that file, you can run the generator to create a new spec file.

```rb
rails generate model_spec mymodel
```
Many gems ship with generators such as this one. In fact, we created a simplified version of the generator [Rspec ships with](https://relishapp.com/rspec/rspec-rails/docs/generators). [FactoryBot has a generator](https://github.com/thoughtbot/factory_bot_rails#generators) for factories. There are many more examples.

The generators in various gems are more sophisticated than the one we created. We could make our generator take arguments or even hook it into existing generators such as `rails scaffold`. Refer to the [Rails generators documentation](https://guides.rubyonrails.org/generators.html#customizing-your-workflow) if you want to learn more.

So generators have the potential to simplify your workflow in an existing application. But can we also use generators to customize setting up a new application?

Enter templates!

## Templates in Ruby on Rails
As the name suggests, templates are files for customizing your application setup. Don't confuse these with the template files that we previously discussed! Under the hood, they are just generators with a specific purpose, as evidenced by the [template API](https://guides.rubyonrails.org/rails_application_templates.html#template-api). While not exactly identical to generators, they are very similar.

If you have an existing template file, you can use it like so:

```zsh
rails new myapp -m mytemplate.rb
```

Rather than specifying a local file, you may also specify a URL. This is especially useful as it allows you to share application templates.

```zsh
rails new myapp -m https://gist.github.com/appsignal/12345/raw/
```

You are not limited to using templates when running rails new either. If you've already set up an app, you can apply templates afterward by executing:

```zsh
rails app:template LOCATION=http://example.com/template.rb
```

Templates can be extremely useful. Who doesn't want to automate adding the same couple of gems and making the same configuration changes every time they create a new app? Creating your own application template is great fun — even if it doesn't save a lot of time in the long run.

## Creating Your Own Template in Rails
We now know about generators and how to use templates. Let's create a simple application template to automate some setup steps.

How you set up your Rails app is very much down to personal preference, but here's an example:

1. Install the dotenv gem.
2. Create a `.env.development` file for the development environment.
3. Adapt the database configuration file to use environment variables.
4. Optionally [install and set up Rspec](https://github.com/rspec/rspec-rails).
Let's create a local file — `mytemplate.rb` — and add `dotenv` using the `gem` command.

```rb
gem 'dotenv-rails', groups: [:development, :test]
```
As we've just added the dotenv gem, let's also create a `.env.development` file to contain our database configuration.

You can create a new file with specific content by using `create_file`. You won't find this in the template or generator documentation, as the method is supplied by Thor. You might also come across the alias `file`. Application templates are evaluated in the context of `Rails::Generators::AppGenerator`, and that's exactly [where the file alias is defined](https://github.com/rails/rails/blob/8f39fbe18a57ae74513edc8561c00a369fe10f08/railties/lib/rails/generators/rails/app/app_generator.rb#L522).

```rb
create_file '.env.development', <<~TXT
  DATABASE_HOST=localhost
  DATABASE_USERNAME=#{app_name}
  DATABASE_PASSWORD=#{app_name}
TXT
```

The `app_name` variable contains the first argument of `rails new`. Using this variable, we can ensure our config file matches the generated application.

Next, let's use our environment variables to connect to the database. We could overwrite the entire `config/database.yml` using the `create_file` command, but let's modify it instead using [inject_into_file](https://guides.rubyonrails.org/generators.html#inject-into-file).

```rb
inject_into_file 'config/database.yml', after: /database: #{app_name}_development\n/ do <<-RUBY
  host: <%= ENV['DATABASE_HOST'] %>
  username: <%= ENV['DATABASE_USERNAME'] %>
  password: <%= ENV['DATABASE_PASSWORD'] %>
  RUBY
end
```
We can use both strings or regex with the `after` argument to specify where to inject content.

Of course, using this kind of configuration only makes sense if a user isn't creating an application with SQLite. You can check for the presence of certain arguments by using the `options` variable. It's best you read the [source code of Rails' app generator](https://github.com/rails/rails/blob/8f39fbe18a57ae74513edc8561c00a369fe10f08/railties/lib/rails/generators/database.rb#L14) to see which options are available.

```rb
if options[:database] != 'sqlite3'
  # Set up env vars and db configuration
end
```

Last but not least, let's also allow users to install Rspec if they want to. There are various methods for taking user input and creating interactive templates. The `yes?` method asks a user for confirmation:

```rb
if yes?('Would you like to install Rspec?')
  gem 'rspec-rails', group: :test
  after_bundle { generate 'rspec:install' }
end
```

We already know the `gem` method, but `generate` and `after_bundle` are new.

As mentioned before, Rspec adds its own generators, and you can call these generators (or any other ones, for that matter) directly from your template. But there is a catch — gems specified with the `gem` method are only installed at the end of the template. Calling `generate` with a generator supplied by such a gem would fail — which is why you should register the command as a callback with `after_bundle`.

Note: Before we wrap up, a quick word about creating or modifying files. We used `create_file` and `inject_into_file`, but there are many other options. You may come across `copy_file` or `template` when reading different templates. I did not mention them here to keep things simple. If you want to create more advanced templates, you should know that these other methods for dealing with files exist.

## The Result: The Final Template in Rails
The final template should look like this:

```rb
# Install dotenv
gem 'dotenv-rails', groups: [:development, :test]

if options[:database] != 'sqlite3'
  # Set up .env.development
  create_file '.env.development', <<~TXT
    DATABASE_HOST=localhost
    DATABASE_USERNAME=#{app_name}
    DATABASE_PASSWORD=#{app_name}
  TXT

  # Modify database.yml
  inject_into_file 'config/database.yml', after: /database: #{app_name}_development\n/ do <<-RUBY
  host: <%= ENV['DATABASE_HOST'] %>
  username: <%= ENV['DATABASE_USERNAME'] %>
  password: <%= ENV['DATABASE_PASSWORD'] %>
  RUBY
  end
end

# Optionally install Rspec
if yes?('Would you like to install Rspec?')
  gem 'rspec-rails', group: :test
  after_bundle { generate 'rspec:install' }
end
```

You can test this particular template by running:

```zsh
rails new myapp -m mytemplate.rb
```

Or:

```zsh
rails new myapp --database=postgresql mytemplate.rb
```

To take advantage of the custom database configuration.

As mentioned, this file is perfectly suited to share as a GitHub gist. Adapt it to suit your needs, upload it, and then share it with your colleagues, friends, and anyone else interested in your custom app template 😉.

## Learn More About Rails Generators and Templates
Needless to say, we've just scratched the surface here.

Everyone has different preferences for writing and developing Rails apps, so there are numerous generators and application templates out there. You can learn a lot about doing specific customizations by reading about them. I recommend [Chris Oliver's Jumpstart](https://github.com/excid3/jumpstart) and [RailsBytes](https://railsbytes.com/), the latter of which is a community-curated collection of templates.

There is also [Thoughtbot's Suspenders](https://github.com/thoughtbot/suspenders), which inspired me to dig deeper into Rails generators and templates. I even wrote my own application template — [Schienenzeppelin](https://github.com/hschne/schienenzeppelin) — which, while not up-to-date, might still provide some inspiration.

## Wrap Up: Get Started with Ruby on Rails Generators and Templates
In this post, we looked into the basics of Rails generators and how they are used. We created our own generators to simplify writing new model specs.

We then delved into templates and learned how you could create a simple template to customize your application setup. This can be a bit of work, but also quite rewarding. If writing your own templates is not for you, there are many existing ones online to choose from!

Original Source:
[Bootstrapping with Ruby on Rails Generators and Templates](https://blog.appsignal.com/2022/05/04/bootstrapping-with-ruby-on-rails-generators-and-templates.html)
