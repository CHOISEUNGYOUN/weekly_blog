# Ruby on Rails 제네레이터와 템플릿을 부트스트래핑하기

레일즈의 [배터리를 포함하는 접근방식(batteries-included, 작업에 필요한 도구들을 가능한 모두 제공하는 방식)](https://en.wikipedia.org/wiki/Batteries_Included)은 가장 큰 장점중에 하나이다. 그 중에서도 레일즈의 제네레이터 덕분에 다른 어떠한 프레임워크도 별 다른 노력없이 처음부터 어플리케이션을 작성할 수 있다.

레일즈를 얼마나 사용했던 간에 필연적으로 제네레이터를 사용해보았을 것이다. 새로운 어플리케이션이 필요하다면 `rails new` 커멘드를 사용한다. 수많은 모델과 뷰를 다루는 스캐폴드를 작성해야한다면 `rails generate scaffold`를 사용하면 된다. 이외에도 다른 여러 제네레이터를 사용하여 빠르게 앱을 작성하고 유연하게 작업을 할 수 있다.

하지만 기본적으로 제공해주는 제네레이터가 모든 문제를 해결해주진 않는다. 아마 제네레이터를 수정하거나 새로운 제네레이터를 만들어야 할 필요가 있을 것이다. 이번 글에서 제네레이터를 어떻게 만들어내는지 살펴보고자 한다. 특히 레일즈에서 템플릿을 활용하여 커스텀 제네레이터를 만드는 것을 중점적으로 다루고자 한다.

## 레일즈 제네레이터란 무엇인가?
파이썬이나 자바스크립트에서 제공해주는 제네레이터 함수와는 다르게 레일즈 제네레이터는 커스텀 [Thor](https://github.com/rails/thor)커멘드를 생성하는 것을 의미한다.

여러 예시가 있겠지만 대표적으로 `rails generate model`을 사용하여 ActiveRecord의 모델을 생성하거나 `rails generate migration`을 사용하여 마이그레이션 파일을 생성하는 커맨드가 있다. 또한 `rails generate generator`를 사용하면 새로운 제네레이터를 생성할 수도 있다.

제네레이터는 서로 다른 제네레이터를 호출할수도 있다. 예를 들어 `rails scaffold`의 경우에는 다른 여러 제네레이터를 함께 호출한다. 이를 통해 파일을 생성하거나 수정 할 수도 있으며, 젬 설치, 특정 rake 작업을 수행할수도 있고 이외 기타 다른 작업들도 수행하게 할 수 있다. 어떻게 동작하는지 이해를 돕기위해 간단한 모델을 생성하는 spec 제네레이터를 작성해보았다.

## Ruby on Rails에서 커스텀 제네레이터를 생성하기

```zsh
rails generate generator model_spec
```

위 커맨드를 입력하면 `/lib/generators/model_spec.`에서 몇가지 새로운 파일들이 생성된다. `lib/generators/model_spec/`에 있는 `model_spec_generator`를 수정하여 모델 spec 파일을 해당 경로에 생성하도록 할 수 있다.

```rb
class ModelSpecGenerator < Rails::Generators::NamedBase
  source_root File.expand_path('templates', __dir__)

  def create_model_spec
    template_file = File.join('spec/models', class_path, "#{file_name}_spec.rb")
    template 'model_spec.rb.erb', template_file
  end
end
```

`template` 커맨드는 `lib/generators/model_spec/templates` 경로에 위치한 템플릿 파일을 찾아 특정 위치에 렌더링 하게 된다. 여기서는 `spec/models` 경로가 될 것이다. 이후 템플릿 파일에 있는 ERB 스타일의 변수를 대체하게 된다.

`source_root`를 지정해놓게되면 해당 제네레이터는 어떤 템플릿 파일을 참조할지 지정할 수 있다. `lib/generators/model_spec/tempates/` 경로에 위치한 `model_spec.rb` 파일 템플릿은 아래와 같은 모습이다.

```rb
require 'rails_helper'

RSpec.describe <%= class_name %>, :model
  pending "add some examples to (or delete) #{__FILE__}"
end
```
위 파일을 생성하게 되면, 제네레이터는 해당 템플릿을 참조하여 새로운 spec 파일을 생성한다.

```zsh
rails generate model_spec mymodel
```
많은 루비 젬들이 위 예시와 비슷한 구조의 제네레이터를 함께 제공하고 있다. 가장 간단한 구조의 제네레이터는 [Rspec 제네레이터](https://relishapp.com/rspec/rspec-rails/docs/generators)와 [FactoryBot 제네레이터](https://github.com/thoughtbot/factory_bot_rails#generators)를 꼽을 수 있다. 이외에도 다른 여러 제네레이터를 제공하는 젬들이 있다.

해당 젬들이 제공하는 제네레이들은 위 예시로 든 제네레이터보다 훨씬 간결하고 실용적이다. [레일즈 제네레이터 문서](https://guides.rubyonrails.org/generators.html#customizing-your-workflow)를 참고하면 개발하면서 필요한 제네레이터를 직접 구현 할 수 있다.

제네레이터는 현재 구현된 앱을 좀 더 간편하게 개발 할 수 있도록 도와준다. 이 뿐만 아니라 새로운 어플리케이션을 만들기 위해 제네레이터를 커스터마이징 하기를 원한다면 지금부터 언급하는 예제를 잘 살펴보자.

## Ruby on Rails 의 템플릿
이름에서 볼 수 있듯이 템플릿은 어플리케이션 설정을 커스터마이징 하기 위한 파일이다. 앞에서 먼저 언급된 템플릿 파일과는 다른 것이니 혼동되지 않았으면 좋겠다. 이 템플릿 파일은 특정 목적을 위한 제네레이터를 구현하기 위해 필요한 요소이다. 더 자세한 내용은
[template API 문서](https://guides.rubyonrails.org/rails_application_templates.html#template-api)를 참고하도록 하자. 제네레이터와는 완전히 동일하지 않으나 목적 그 자체는 비슷하다.

템플릿 파일이 있다면 아래 커멘드를 사용하면 된다.

```zsh
rails new myapp -m mytemplate.rb
```

로컬 파일을 가리키는 대신 URL을 사용할 수도 있다. 이는 어플리케이션 템플릿을 공유하고자 할 때 특히 유용하다.

```zsh
rails new myapp -m https://gist.github.com/appsignal/12345/raw/
```

템플릿 파일은 새로운 레일즈 어플리케이션을 만들때만 유용한 것이 아니라 이후에 템플릿을 적용할 때에도 유용하다.

```zsh
rails app:template LOCATION=http://example.com/template.rb
```

위와 같이 템플릿은 매우 유용한 툴이다. 새로운 젬을 자동으로 추가하고 새로운 어플리케이션을 만들때마다 동일한 환경설정을 구성해 주는데 원하지 않는 사람이 있을까? 커스텀 어플리케이션 템플릿을 만드는 것은 언제나 즐거운 일이다.

## Rails 에서 커스텀 템플릿 만들기
위에서 제네레이터가 어떤 역할을하고 템플릿을 어떻게 사용하는지 배우게 되었다. 이제는 간단한 어플리케이션 템플릿을 만들어 몇가지 설정을 자동화 하는 것을 살펴보자.

레일즈 어플리케이션을 셋업하는 것은 개개인별로 다를 수 있지만 이번 예제에서는 아래의 방법으로 진행하고자 한다.

1. dotenv 젬을 설치한다.
2. 개발환겨을 위한 `.env.development` 파일을 생성한다.
3. 환경변수를 적용하기 위해 데이터베이스 설정을 가져온다.
4. 필요하다면 [Rspec 을 설치하고 설정](https://github.com/rspec/rspec-rails) 한다.
`mytemplate.rb` 로컬 파일을 만들고 `dotenv` 젬을 추가하자.

```rb
gem 'dotenv-rails', groups: [:development, :test]
```

`dotenv`젬 파일을 추가했다면 `.env.development` 파일도 생성해서 데이터베이스 설정도 해당 파일에 기록하자.

이제 `create_file` 커멘드를 사용해서 특정 컨텐츠를 포함한 새로운 파일을 생성 할 수 있다. 이 메소드는 `Thor`에 의해 지원되고 있기 때문에 템플릿이나 제네레이터 문서에서는 찾아볼 수 없을 것이다. 어플리케이션 템플릿은 `Rails::Generators::AppGenerator` 컨텍스트를 참조하여 평가를 하게되는데 이 때 [파일 별칭(file alias)](https://github.com/rails/rails/blob/8f39fbe18a57ae74513edc8561c00a369fe10f08/railties/lib/rails/generators/rails/app/app_generator.rb#L522)이 정의된다.

```rb
create_file '.env.development', <<~TXT
  DATABASE_HOST=localhost
  DATABASE_USERNAME=#{app_name}
  DATABASE_PASSWORD=#{app_name}
TXT
```

`app_name` 변수는 `rails new` 의 첫번째 인자값을 가지고 있다. 이 변수를 사용해서 설정파일이 생성된 어플리케이션과 일치하는지 보장할 수 있게 된다.

다음으로 데이터베이스에 연결하기 위해 환경변수들을 이용해보자. `create_file` 커멘드를 사용하여 `config/database.yml` 파일 전체를 다시 쓸 수 있지만 그 대신 [inject_into_file](https://guides.rubyonrails.org/generators.html#inject-into-file)을 사용하여 변경을 해보자.

```rb
inject_into_file 'config/database.yml', after: /database: #{app_name}_development\n/ do <<-RUBY
  host: <%= ENV['DATABASE_HOST'] %>
  username: <%= ENV['DATABASE_USERNAME'] %>
  password: <%= ENV['DATABASE_PASSWORD'] %>
  RUBY
end
```
문자열 또는 정규표현식의 `after` 인자를 사용하여 어디에 컨텐츠를 주입할지 명시 할 수 있다.

물론 이런 설정은 SQLite를 이용하여 어플리케이션을 만들지 않는 이상 필요하진 않을 것이다. 특정 인자의 존재 여부를 `options` 변수를 사용하여 확인할 수 있다. 어떤 옵션들이 사용 가능한지를 확인하고 싶다면 [레일즈 어플리케이션 제네레이터의 소스코드를 확인하는 것이 가장 좋다.](https://github.com/rails/rails/blob/8f39fbe18a57ae74513edc8561c00a369fe10f08/railties/lib/rails/generators/database.rb#L14)

```rb
if options[:database] != 'sqlite3'
  # Set up env vars and db configuration
end
```

마지막으로 필수는 아니지만 원하는 경우 Rspec을 설치하도록 허용해보자. 사용자 입력값을 받고 대화형 템플릿(interactive template)을 만드는 방법에는 여러 가지가 있다. 여기서 `yes?` 메소드는 사용자의 확인을 받는다.

```rb
if yes?('Would you like to install Rspec?')
  gem 'rspec-rails', group: :test
  after_bundle { generate 'rspec:install' }
end
```

`gem` 메소드는 이미 알고 있지만, `generate` 와 `after_bundle`은 우리가 알지 못한 새로운 메소드들이다.

앞서 언급했듯이 Rspec은 내부에 제네레이터가 따로 있기 때문에 템플릿에서 해당 제네레이터를 바로 불러올 수 있다. 하지만 여기서 조건은 `gem` 메소드로 지정한 젬들만 템플릿에 추가된다는 점이다. 해당 젬을 참조하지 않고 `generate`를 제네레이터를 이용하여 호출하게 되면 실패 할 것이다. 이로 인해 `after_bundle` 콜백을 이용하여 해당 커맨드를 입력해야 한다.

*참고: 마무리 하기 전에 파일 생성 또는 수정을 할 시 살펴보아야 할 점은 `create_file` 과 `inject_into_file` 이외에도 다른 많은 옵션이 있다는 점이다. 아마 다른 템플릿에 관한 글을 보았다면 `copy_file`이나 `template` 또한 존재함을 알고 있을 것이다. 이 글에서는 해당 내용도 추가하면 너무 복잡해져서 추가하지 않았다. 좀 더 어려운 템플릿을 작성하기를 원한다면 템플릿 문서나 소스코드를 참고하는 것이 좋다.*

## 레일즈 템플릿 결과물
언급한 템플릿 작성 방식을 모두 사용한다면 아래와 같은 코드를 작성했을 것이다.

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

위 템플릿을 아래 커멘드로 테스트 할 수 있다.

```zsh
rails new myapp -m mytemplate.rb
```

또는

```zsh
rails new myapp --database=postgresql mytemplate.rb
```

앞서 언급했듯이 지금까지 작성한 파일은 Github gist에 공유되어 있다. 당신의 필요에 맞게끔 수정 및 업로드 하여 다른 팀 동료나 친구 또는 커스텀 템플릿을 필요로 하는 사람들에게 공유해도 좋다.

## Rails 제네레이터와 템플릿에 대해 좀 더 알고 싶다면...
말할 필요도 없이 우리가 지금까지 했던 과정은 그저 기초에 불과하다.

각자 제각각의 취향에 맞춘 제네레이터와 어플리케이션 템플릿이 여기저기 공유되어 있다. 필요에 따라 어떻게 구현했는지 각 제네레이터와 템플릿을 읽으며 습득하면 된다. 특히 [Chris Oliver's Jumpstart](https://github.com/excid3/jumpstart) 와 [RailsBytes](https://railsbytes.com/)를 추천한다. 후자는 템플릿을 공유한 커뮤니티이다.

또한 [Thoughtbot's Suspenders](https://github.com/thoughtbot/suspenders) 추천하는데 이는 레일즈 제네레이터와 템플릿에 관해 좀 더 깊이 파고들게 하는 원동력이 되었다. 필자(여기서는 원작자) 또한 [Schienenzeppelin](https://github.com/hschne/schienenzeppelin)이라는 템플릿을 작성하였다. 이 템플릿은 더이상 최신은 아니지만 그래도 통찰 정도는 줄 수 있지 않을까 생각한다.


Original Source:
[Bootstrapping with Ruby on Rails Generators and Templates](https://blog.appsignal.com/2022/05/04/bootstrapping-with-ruby-on-rails-generators-and-templates.html)
