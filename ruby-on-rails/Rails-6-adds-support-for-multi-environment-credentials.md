## Rails 6 : 다중 환경 인증 지원 기능 추가

### 배경
보통 어플리케이션을 구성할 때 API 키나 시크릿 키 등의 다양한 시크릿키와 인증키들이 존재한다. 우리는 이러한 키들을 보편적이면서도 안전하게 관리 해야한다.

Rails 5.1 에서는 이러한 인증을 관리 할 수 있는 [`secrets`](https://github.com/rails/rails/pull/28038) 기능이 추가되었다.
Rails 5.2 에서는 암호화된 시크릿 키들이 다른 인증키들과 함께 관리하기 어렵기 때문에 `secrets` 기능이 `credentials` 로 대체되었다.

이러한 인증키들을 관리하기 위해 아래와 같은 파일들이 필요했었다.

* `config/credentials.yml.enc`
* `config/master.key`

`config/credentials.yml.enc` 는 인증키들을 저장하는 암호화된 파일이다. 암호화 처리 되어있기 때문에 `git`과 같은 VCS(Version Control System, 버전 관리 시스템)에 안전하게 기록하고 관리 할 수 있다.

`config/master.key` 는 `config/credentials.yml.enc` 를 복호화하기 위해 사용되는 `RAILS_MASTER_KEY` 를 가지고 있다. 이러한 이유로 이 파일은 VCS에 기록하고 관리를 하면 안된다.

### 인증키를 가져오는 방법
`config/credentials.yml.enc` 는 암호화 되어있기 때문에 해당 파일에 바로 접근하여 읽어올 수 없다. 대신 Rails 에서 암호화 및 복호화를 추상화 하여 제공해주는 기능을 사용 할 수 있다.

### 인증키 추가 및 변경 방법
인증키를 아래 명령어를 `shell` 에서 실행하면 편집 가능하디.

```shell
$ EDITOR=vim rails credentials:edit
```

이 명령어를 실행하면 복호화 된 인증키 파일이 `vim` 에디터를 통해 열린 것을 확인 할 수 있다.
여기서 인증키를 `YAML` 형식에 맞추어 업데이트 할 수 있다. 아래 예제를 추가하고 저장해보자.

```yaml
aws:
  access_key_id: 123
  secret_access_key: 345
github:
  app_id: 123
  app_secret: 345
secret_key_base:
```

저장을 하면 Rails에 저장되어 있는 마스터 키를 사용하여 다시 암호화 한다.

기본 에디터가 지정되어있지 않고 어떤 에디터를 실행할지 지정하지 않는다면, 아래와 같은 에러메세지가 나타난다.

```shell
$ rails credentials:edit
No $EDITOR to open file in. Assign one like this:

EDITOR="mate --wait" bin/rails credentials:edit

For editors that fork and exit immediately, it's important to pass a wait flag,
otherwise the credentials will be saved immediately with no chance to edit.
```

### 인증키를 읽어오는 방법
인증키를 읽어오는 방법은 아래 예시와 같다.

```rb
> Rails.application.credentials.config
#=> {:aws=>{:access_key_id=>"123", :secret_access_key=>"345"}, :github=>{:app_id=>"123", :app_secret=>"345"}}
> Rails.application.credentials.github
#=> {:app_id=>"123", :app_secret=>"345"}
> Rails.application.credentials.github[:app_id]
#=> "123"
```

### Rails 6 이전에 사용하던 다중 환경 인증키 관리 방법
Rails 6 이전에는 Rails 자체적으로 다중 환경 인증키를 관리하는 방법이 제공되지 않았다. 각각 환경변수에 맞추어 인증키를 관리하는 방법은 있었으나 사용하는 개발자 스스로 해당 키들을 지정해주어야 했다.

Rails 6 이전에는 아래와 같이 하나의 파일에 환경별 인증키를 관리했었다.

```yaml
development:
  aws:
    access_key_id: 123
    secret_access_key: 345

production:
  aws:
    access_key_id: 1f3649fe-ebbd-11e9-81b4-2a2ae2dbcce4
    secret_access_key: 203060d3a5456fa6cd2da3c958001440    
```

이렇게 하고 아래와 같이 인증키를 구분해서 사용 할 수 있었다.

```rb
> Rails.application.credentials[Rails.env.to_sym][:aws][:access_key_id]
#=> "123"
```

### 이 방법의 문제점

* 해당 앱엔 모든 개발팀 멤버들이 접근 할 수 있는 마스터키가 하나만 존재한다. 이 말은 개발팀 누구나 프로덕션 환경에 접근 가능하다는 것이다.
* 각 환경별로 사용되는 인증키를 각각 명시 해줘야 한다.

다른 방법으로는 각 환경 조건에 맞는 인증키 보관 파일을 생성하는 것이다. 예를 들면, 스테이징 환경용 `config/staging.yml.enc` 를 만든다던지, 프로덕션 환경용 `config/production.yml.enc` 를 만드는 것이다. 해당 config 파일을 읽어오기 위해선 [Rails 5.2 에서 지원하는](https://github.com/rails/rails/commit/68479d09ba6bbd583055672eb70518c1586ae534) 여러 인증키 파일을 각각 암호화 및 복호화를 하는 기능을 사용해야 한다.

이러한 접근법은 각각의 환경에 맞는 인증키 파일을 관리 해야하기 때문에 더 많은 상용구 코드(컴퓨터 프로그래밍에서 상용구 코드 또는 상용구는 수정하지 않거나 최소한의 수정만을 거쳐 여러 곳에 필수적으로 사용되는 코드를 말한다. 출처-위키백과)를 작성해야 한다.

### Rails 6에서 관리하는 방법
Rails 6에서는 이러한 문제를 해결하기 위해 [다중 환경 인증키 관리 기능](https://github.com/rails/rails/pull/33521)을 추가했다.

이 기능은 각 환경에 맞춘 인증키들을 손쉽게 관리 할 수 있도록 각 환경별로 암호화 키를 가지고 있다.

### 전역에서 사용 가능한 인증키
위 PR에서 추가된 기능은 역방향으로도 동작한다. 특정 환경에 대한 인증키가 없는 경우 Rails는 자동으로 전역에 선언된 인증키와 마스터키를 아래 파일에서 읽어온다.

* `config/credentials.yml.enc`
* `config/master.key`

필자의 개발팀은 전역 환경 변수는 개발 환경이나 테스트 환경에서만 사용한다. `config/master.key` 파일은 팀원 전체와 공유한다.

### 프로덕션 환경용 인증키 만들기
프로덕션 환경용 인증키를 만들기 위해선 아래의 명령어를 실행한다.

```shell
$ rails credentials:edit --environment production
```

위 코드는 아래의 작업을 수행한다.

* `config/credentials/production.key` 파일이 없는 경우 생성한다. VCS에 저장 하면 안된다.
* `config/credentials/production.yml.enc` 파일이 없는 경우 생성한다. 해당 파일은 VCS에 저장한다.
* 프로덕션용 인증키 파일을 복호화 하고 기본 에디터로 실행한다.

필자의 팀은 특정 멤버에게만 프로덕션 환경용 `production.key` 파일을 공유한다.

아래 예제코드를 추가하고 저장해보자!

```yaml
aws:
  access_key_id: 1f3649fe-ebbd-11e9-81b4-2a2ae2dbcce4
  secret_access_key: 203060d3a5456fa6cd2da3c958001440
```

이와 비슷하게 스테이징 환경용 인증키 파일을 생성하고 관리 할 수 있다.

### Using the credentials in Rails
### Rails 애서 인증키 사용하기
Rails 는 각 환경에 맞춘 인증키 파일을 자동으로 읽어온다. 특정 환경용 인증키는 전역에 선언된 인증키를 오버라이딩 한다. 해당 환경용 인증키가 없는 경우에는 전역 환경으로 선언된 인증키 파일을 읽어온다.

개발 환경 예시:

```rb
$ rails c
> Rails.application.credentials.config
#=> {:aws=>{:access_key_id=>"123", :secret_access_key=>"345"} }}
> Rails.application.credentials.aws[:access_key_id]
#=> "123"
```

프로덕션 환경 예시:

```rb
$ RAILS_ENV=production rails c
> Rails.application.credentials.config
#=> {:aws=>{:access_key_id=>"1f3649fe-ebbd-11e9-81b4-2a2ae2dbcce4", :secret_access_key=>"203060d3a5456fa6cd2da3c958001440"}}
> Rails.application.credentials.aws[:access_key_id]
#=> "1f3649fe-ebbd-11e9-81b4-2a2ae2dbcce4"
```

### 환경변수에 암호화 키 저장하기

또한 Rails는 각 환경변수에 지정된 암호화 키를 자동으로 확인하고 사용한다. 범용적으로 사용되는 환경변수인 `RAILS_MASTER_KEY` 또는 `RAILS_PRODUCTION_KEY`와 같이 특정 환경용 변수를 사용할 수 있다.

이런 환경 변수들이 선언되어 있으면 `*.key` 와 같은 별개의 암호화 키 파일을 만들 필요가 없다. Rails가 자동으로 해당 환경 변수를 읽어와서 암호화/복호화 작업을 실행한다.

이러한 환경 변수들은 `Heroku` 나 다른 비슷한 플랫폼에서도 사용 가능하다.

```shell
# Setting master key on Heroku 
heroku config:set RAILS_MASTER_KEY=`cat config/credentials/production.key`
```

Original Source:
[Rails 6 adds support for multi environment credentials](https://blog.saeloun.com/2019/10/10/rails-6-adds-support-for-multi-environment-credentials)
