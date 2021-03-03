# 더 쉬운 Rails 5.2 자격 증명(credentials) 사용법과 어플리케이션 한정 설정 방법

지금 알다시피 Rails 5.2 버전에서는 `Credentials`라는 새로운 기능이 추가되었다. DHH는 해당 버전 PR에서 다음과 같이 말했다.

> 이 새로운 파일은 플랫 포멧 형식을 지니며 `secrets.yml` 파일처럼 각 환경마다 분리되지 않는다. 대부분의 자격증명 요소들은 실제 배포환경에서만 사용된다. 다른 환경에 대한 동일한 자격증명이 필요한 경우 수동으로 직접 적용 할 수 있다.

*참고: 이러한 불편함은 [Rails 6.0](https://github.com/rails/rails/pull/33521)에서 수정되었다.*

어쨌든 Rails 5.2 에서 보통 쓰이는 방법은 아래 예시와 같이 직접 환경변수로 적용하는 것이다.

```rb
development:
  facebook_app_id: '...'
  facebook_app_secret: '...'
  facebook_app_namespace: '...'
  stripe_publishable_key: '...'
  stripe_secret_key: '...'
  stripe_signing_secret: '...'

production:
  facebook_app_id: '...'
  facebook_app_secret: '...'
  facebook_app_namespace: '...'
  stripe_publishable_key: '...'
  stripe_secret_key: '...'
  stripe_signing_secret: '...'
```

사용법은 아래와 같다.
```rb
Rails.application.credentials[Rails.env.to_sym][:facebook_app_secret]
```
꽤 길고 가독성이 좋지 않다. 아마 당신의 어플리케이선에서 이런식으로 자격증명을 많이 하는 경우는 없을 것이다. 그래도 47자나 되는 Rails 코드에서 단지 22자만 필요해 보이는건 사실이다. 필자의 생각에는 좋지 못한 비율이다.

여기서 필자가 제안하는 방법은 단순하지만 이런식으로 사용 할 것을 제안한 글은 한번도 본적이 없다. (아마도 충분히 검색해보지 못했을 수도 있다. 만약 그렇다면 미리 사과한다.)

`config/application.rb`를 열어서 다음과 같이 `credentials`를 추가하기만 하면 된다.

```rb
require_relative 'boot'

require 'rails/all'

# Gemfile에 있는 모든 Gem들을 require 해야한다.
# :test, :development, 또는 :production 으로 제한한다.
Bundler.require(*Rails.groups)

module YourApp
  class Application < Rails::Application
    # 원래 생성된 Rails 버전에 맞추어 설정 기본값을 초기화 한다.
    config.load_defaults 5.2

    # config/envirnments/* 에 위치한 설정값이 여기에 표기된 설정보다 우선순위에 있다.
    # 어플리케이션 설정은 config/initializers 에서 가저온다.
    # config/initializers 에 위치한 모든 .rb 파일들은 프레임워크나 Gem이 실행될때 자동으로 불러온다.
  end

  def self.credentials
    @credentials ||= Rails.application.credentials[Rails.env.to_sym]
  end
end
```

그럼 어떤점이 바뀌었는지 한번 살펴보자.

```rb
# Before:
Rails.application.credentials[Rails.env.to_sym][:facebook_app_secret]

# After
YourApp.credentials[:facebook_app_secret]
```

이러한 접근법은 Rails 특유의 사용법을 추상화 하면서도 코드를 좀 더 명확하게 해준다. 추가로 유연성 또한 제공한다. Rails 자격증명을 다른 요소로 대체하고 싶다면 자격증명을 다른 요소와 합칠 수 있다. 이는 `[Rails.env.to_sym]` 를 제거함으로써 Rails 6의 자격증명 시스템으로 손쉽게 업그레이드 할 수 있다.

이것만 보아도 근사하지만, 이 글의 제목에서 보았듯이 **"어플리케이션 한정 설정 방법"** 에 대하여 다뤄보고자 한다. Rails 사용법을 한결 쉽게 할 수 있는 또 한가지 방법에 대하여 설명하겠다.

Rails 어플리케이션이 점점 복잡해짐에 따라 개별 환경에 맞춘 다양한 설정값을 추가하고 싶을 것이다. 이는 비밀 키나 자격증명이 아닌 단순히 사용자 증명 환경 변수를 어플리케이션 전체에 적용하는 것을 말한다. Rails 문서에는 정확히 해당 부분에 대한 내용 또한 다루고 있다. <br>
참고 : [사용자 정의 환경 변수](https://guides.rubyonrails.org/v5.2/configuring.html#custom-configuration)

Rails 문서에 따르면 아래 두가지 강력한 옵션을 제시하고 있다.

1. `config.x` 네임스페이스
2. `config_for` 메소드

정말 멋진 기능이 아닐 수 없다! 위에서 소개한 자격증명 적용 방식과 사용자 정의 환경 변수를 함께 사용해보자!

Step 1: `config/app.yml` 파일을 생성한다.

```rb
shared: &shared
  facebook_url: 'https://www.facebook.com/mysite'
  twitter_url: 'https://twitter.com/mysite'

development:
  <<: *shared
  domain: 'localhost:3000'
  devise_mailer_sender: 'no-reply@localhost'
  orders_mailer_sender: 'no-reply@localhost'

production:
  <<: *shared
  domain: 'mysite.com'
  devise_mailer_sender: 'no-reply@mysite.com'
  orders_mailer_sender: 'orders@mysite.com'
```

Step 2: `config/application.rb` 파일에 `config.x` 선언하고 `app.yml`에 있는 내용들을 추가한다.

```rb
# 어플리케이션 한정 환경 변수
config.x = config_for(:app).with_indifferent_access
```

Step 3: `config/appication.rb` 파일에 `config` 메소드를 추가한다.

```rb
  def self.config
    @config ||= Rails.configuration.x
  end
```

`config/application.rb` 파일은 다음과 같을 것이다.

```rb
require_relative 'boot'

require 'rails/all'

# Gemfile에 있는 모든 Gem들을 require 해야한다.
# :test, :development, 또는 :production 으로 제한한다.
Bundler.require(*Rails.groups)

module YourApp
  class Application < Rails::Application
    # 원래 생성된 Rails 버전에 맞추어 설정 기본값을 초기화 한다.
    config.load_defaults 5.2

    # config/envirnments/* 에 위치한 설정값이 여기에 표기된 설정보다 우선순위에 있다.
    # 어플리케이션 설정은 config/initializers 에서 가저온다.
    # config/initializers 에 위치한 모든 .rb 파일들은 프레임워크나 Gem이 실행될때 자동으로 불러온다.

    # 어플리케이션 한정 환경 변수
    config.x = config_for(:app).with_indifferent_access
  end

  def self.config
    @config ||= Rails.configuration.x
  end

  def self.credentials
    @credentials ||= Rails.application.credentials[Rails.env.to_sym]
  end
end
```

위에 적용한 사용자 정의 환경 변수를 아래와 같이 사용할 수 있다.

```rb
YourApp.config[:devise_mailer_sender]
```

정말 멋지지 않는가?

Original Source:
[Easier usage of Rails 5.2 credentials and app-specific configuration](https://dev.to/svyatov/easier-usage-of-rails-52-credentials-and-app-specific-configuration-461g)
