## DHH는 Rails Controller를 어떻게 정리할까?

우리의 대장이자 구세주인(?) DHH는 최근 [Full Stack Radio 인터뷰](https://fullstackradio.com/32) 에서 가장 최신 버전의 [베이스캠프](https://basecamp.com/) 앱에서 어떻게 컨트롤러는 정리했는지 이야기했다. 이 글에서 그 내용을 조금 수정 보완해두었다.

DHH가 최근에 깨달은것이 있다면 REST 방식을 고수하기 위해 매번 새로운 컨트롤러를 만드는 기초적인 방법이 항상 훨씬 더 나은 결과를 가져다 준다는 것이었다. 구성해둔 컨트롤러의 상태를 볼 때마다 너무 적게 만든게 아닌가 하는 후회를 했었다. 너무 많은것들을 하나의 컨트롤러에 담으려고 했었기 때문이다.

그래서 이번 베이스캠프 3에서는 특정 조건에 따라 화면이 달라지는 필터를 것처럼 상위/하위 리소스에 상관없이 컨트롤러를 분리했다. 예를 들어 드롭다운 버튼에 몇가지 필터를 적용해두면 이는 그때마다 다른 상태가 된다. 우리는 대게 이러한 상태들을 하나로 모아 새로운 컨트롤러를 만들었다.

이러한 휴리스틱 관점에서 내가 내린 결정은 바로 기본 다섯가지 액션이나 기본적으로 내장되어있는 REST 기능이 아닌 메소드가 필요한 경우 새로운 컨트롤러를 만들고 호출하는 것이다!

예를 들어 우리가 `InboxController` 를 만들고 여기서 해당 inbox에 있는 모든것을 전달한다고 생각해보자. 아마 아직 대기중인 데이터를 보여주는 액션이 필요하다고 생각할 것이다. 보통 아래 예제처럼 `pending` 이라는 액션을 컨트롤러에 추가할 것이다. 

```rb
class InboxesController < ApplicationController
  def index
  end

  def pendings
  end
end
```

매우 일반적인 패턴이다. DHH 또한 이러한 패턴을 즐겨 사용했다.

하지만 이번 프로젝트에서는 `index` 액션 하나만 존재하는 `Inboxes::PendingsController` 를 새로 만들었다.

```rb
class InboxesController < ApplicationController
  def index
  end
end
```

```rb
class Inboxes::PendingsController < ApplicationController
  def index
  end
end
```

이렇게 사용하면 각 컨트롤러에 권한을 위임하고 그 안에서만 특정 조건의 필터들을 적용 할수 있다.

이러한 방식을 이용하면 네임스페이스 내부에서 컨트롤러를 쉽게 확장 할 수 있다. 예를 들어 `MessagesController` 가 있다고 생각해보자. 네임스페이스를 이용하면 `Messages::DraftsController` 나 `Messages::TrashesController` 처럼 네임스페이스 규칙 내에서 하위 컨트롤러를 쉽고 명확하게 확장할수 있다. 정말 놀라운 발견이었다!

그래서 DHH는 컨트롤러에서 기본적으로 제공하는 `index`, `show`, `new`, `edit`, `create`, `update`, `destroy` 등의 기본 CRUD 액션만을 사용할 것을 권장했다. 다른 액션이 필요하면 해당 기능을 수행하는 컨트롤러를 새로 만들면 된다.(이 또한 기본으로 지정된 CRUD 액션만 사용한다.)

### 필자가 느낀점

여기서부턴 DHH의 생각이 아닌 필자의 생각이다. 다른 의견을 제시해도 좋다. 

어쨌든 DHH 방식처럼 컨트롤러를 구성하는 방식에 매우 만족하고 있다. 위에 그가 언급한 예시는 단순히 필터링을 위한 액션이었고, 다른 시선에서 본다면 분명히 오버엔지니어링 일수도 있다. REST 구조 내에서 필터링을 하는 기능은 보통 쿼리 파라미터를 이용하여 구성하기 때문에 (예: `GET /inboxes?state=pending`) 코드가 심플하고 간결하다면 기존의 방식을 따를 것이다. (하지만 이게 길고 복잡해지거나 너무 많은 믹스인 액션이 혼재된다면 DHH의 방식을 따를 것이다.)

그래도 컨트롤러를 분리한다는 그의 관점은 충분히 동의한다. 그에 대한 이유 몇가지를 언급해보겠다.

### 좀 더 간결한 코드를 작성할 수 있다.

이 방법을 사용하면 필요한 만큼 컨트롤러를 계속 만들 수 있다. 우리가 만든 컨트롤러에 기본 CRUD 액션만 있으면 심플하고 간결한 컨트롤러를 구성할 수 있다. 컨트롤러에서 덜 가공된 데이터를 `index` 나 `show` 같은 액션을 통해 불러오지 않아도 된다.

컨트롤러 분리 전략은 컨트롤러 기능이 점점 비대해질때 빛을 발한다. 컨트롤러가 점점 비대해지더라도 오직 기본 CRUD 액션만 있다면 그냥 기능들을 분할하여 새로운 컨트롤러에 재배치 하기만 하면 되기 때문이다.

예를 들어 필자가 다니고 있는 회사의 컨트롤러중 가장 복잡한 컨트롤러는 다음과 같다.(우리는 가벼운 모델, 뚱뚱한 컨트롤러 전략-YMMV 을 사용하고 있다.) 이 전략을 사용한다면 제품 구매 API 로직을 아래와 같이 구성할 수 있다.

```rb
class Api::V1::PurchasesController < Api::V1::ApplicationController

  rescue_from Stripe::StripeError, with: :log_payment_error

  def create
    load_product
    load_device
    load_or_create_user

    create_order
    create_payment
    authorize_payment
    confirm_address

    render json: @order, status: :created
  end

  private

  def load_product
    @product = Product.find_by!(uuid: params[:product_id])
  end

  # …

end
```

보시다시피 여기에는 기본 액션중 하나인 `create` 액션 하나만 존재한다. 불완전한 추상화를 모델에 추가하지도 않았고, 서비스 클래스, 옵저버 패턴등 다른 요소를 사용하지도 않았다. 필요한 모든것들은 여기 컨트롤러에 깔끔하게 정리해두었다. 여기에 구성된 메소드들을 이해하기위해 다른 파일들을 쓸데없이 왔다갔다 하지 않아도 된다. 딱 이 컨트롤러 파일 하나만 에디터로 열어보면 이해된다. 컨트롤러는 웹 어플리케이션 코드의 진입로와 같기 때문에 코딩을 하게된다면 반드시 열게 되어있는데, 이 파일 하나면 열면 모든것이 해결된다.

아마 이 글을 읽고 있는 당신은 "이게 뭐야... 여기에 모든 메소드를 집어넣었어? 이거 진짜 장난아니게 길겠네?" 라고 생각할수도 있다. 안타깝게도 이 컨트롤러는 단지 144줄에 불과하다.

그리고 이 컨트롤러가 우리 앱에서 가장 복잡한 컨트롤러다. 진짜 최악중에 최악인 케이스다. 아마 어떻게 코드를 분리하는지에 따라 다르겠지만 이렇게 분리하면 보기 깔끔해진다. 다른 "복잡한" 컨트롤러들은 이것보다 훨씬 간결하다. 6줄에서 103줄 정도 되는 컨트롤러들이 존재한다. 평균으로 계산해보자면, 컨트롤러당 **15줄** 정도 되는 것 같다.(참고로 필자 회사의 앱에는 150개가 넘는 컨트롤러가 있다.)

명심해라. 컨트롤러마다 요청과 관련된 코드는 아주 조금밖에 없으면서 200줄이 넘는 코드가 존재하고 나머지 로직들을 다른 서비스 패턴이나, 옵저버 패턴 또는 모델에서 처리해주는 프로젝트를 작업하고 있다면 그것이이야 말로 끔찍한 참사다. 컨트롤러를 기능별로 분리한다면 [rule of 3](https://en.wikipedia.org/wiki/Rule_of_three_(computer_programming)) 를 따르는 것 처럼 간결하게 코드를 구성할 수 있다. 그리고 [중복이 잘못된 추상화 보다 훨씬 낫다](https://sandimetz.com/blog/2016/1/20/the-wrong-abstraction) 라는 말처럼 코드를 어렵게 만들지도 않는다. 필자는 개인적으로(DHH가 말했듯이) [DRY- Don't Repeat Yourself 똑같은 로직을 반복하지 마라](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) 나 [SRP - Single Responsibility Principle - 단일 책임 원칙](https://en.wikipedia.org/wiki/Single-responsibility_principle) 은 너무 과대평가 되었다고 생각한다.

### 코드를 더 균일하게 만들어준다.

알다시피 하나의 컨트롤러에 많은 CRUD 액션을 추가하는 것은 멋져보인다. 하나의 기묘한 액션을 찾아내기 위해 끝없이 스크롤을 내려가며 분석할 이유가 없어진다. Route에 커스텀으로 추가한 컨트롤러가 있는지, 그것이 어떻게 동작하는지 알아내지 않아도 된다.

필자는 반복적인 작업을 할때 한번에 와닿지 않는 코드를 작성하는 것을 좋아하지 않는다.[놀람 최소화 원칙](https://en.wikipedia.org/wiki/Principle_of_least_astonishment) 균일하면서 예측가능한, 그리고 [설정보다 관습](https://rubyonrails.org/doctrine/#convention-over-configuration)에 따른 방식이 좋다고 생각하며, 이것 때문에 아직도 다른 프레임워크보다 Rails를 사용하는것을 선호한다. 모든것이 동일한 규칙으로 정리되기 때문에 반복적인 작업을 수행할 때 훨씬 더 적은 시간을 들이고도 훨씬 더 빠르게 결과물을 내놓을 수 있다. 이는 스타트업 세계에서 엄청 중요한 문제이다.

이론적으로 생각해보면 이는 곧 하나의 코드베이스에서 다른 코드베이스로 옮길때 완벽하게 효율적으로 이전 할 수 있음을 의미한다. 실제로 많은 개발자들이 "끔찍하게 빌드된" Rails 앱들을 현업에서 많이 마주하게 된다. 어떤 회사는 [옵저버 패턴](https://github.com/rails/rails-observers)을 사용하기도 하고(필자의 취향은 아니다.), 다른 또 회사는 [Trailblazer](https://github.com/trailblazer/trailblazer) 패턴을 Rails에 추가해서 사용하기도 하고(이것도 역시 취향은 아니지만 좀 흥미로운 아이디어는 있음.), 다른 회사는 또 다른 툴을 사용하거나 다른 패턴을 사용하기도 한다.

이러한 기조는 보통 개발자들이 순수한 Rails의 기능이 "구조적으로 결함이 있다" 라고 생각하기 때문이다. 하지만 필자는 다르게 생각한다. 이 모든것은 단순히 컨트롤러를 잘게 분리하고 기본 CRUD 액션을 충실히 잘 사용하면 해결된다고 생각한다. 간결하고 새로이 합류하는 주니어 개발자들이 이해하기도 쉽기 때문이다.

Rails는 이러한 컨트롤러 분리 휴리스틱을 이용하면 좀 더 효율적으로 앱을 구성할수 있다. [Rails 문서](https://guides.rubyonrails.org/routing.html#non-resourceful-routes)에서도 route에서 제공하는 기본 resouce만 사용하는것을 권장한다고 명확하게 가이드 하고 있다. 이 내용은 [Rails 문서](https://guides.rubyonrails.org/routing.html#resource-routing-the-rails-default)에서 아주 옛날부터 친절하게 설명해두었다. 한번이라도 제대로 읽었다면, 커스텀 액션을 추가하는 것이 Rails 에서 추구하는 방향이 아니라는 것은 지나가던 개도 알수 있었을 것이다. 컨트롤러를 분리하는것이야 말로 이 문서에서 말해주고 싶은 이야기였을 것이다.

### REST가 진정 무엇인지 이해할 수 있게 해준다.

많은 사람들이 REST 방식이 규칙적이고 간결하기 때문에 선호한다. 진짜 RESTful한 API가 무엇인지 제대로 이해한다면 필자가 무슨 말을 하고 싶은지 바로 이해할 것이다. 비즈니스 로직은 어플리케이션마다 상이하기 때문에 그것들이 무엇인지 이해하고 어떻게 사용해야 하는지 바로 이해 할 수 있어야 한다. [Stripe](https://stripe.com/docs/api#create_charge) 에서 결제 API를 사용하든 [Twillo](https://www.twilio.com/docs/sms/api/message-resource)에서 SMS 자동 발신 API를 사용하든 [Github](https://docs.github.com/en/rest/reference/repos#get) 에서 저장소를 받아오는 API를 사용하든 말이다.

이걸 이해하기 위해선 Rails의 action을 생각하기 이전에 REST 규칙에 따라 어떤 명사를 사용할지 먼저 곰곰히 생각해보아야 한다. 당신이 직접 "지불" 하지 않는다. 단지 "결제를 만들" 뿐이다. 당신이 직접 "당신 계좌에 입금하지 않는다". 계좌에 입금 하도록 "만들" 뿐이다. 처음엔 이해하기 어려울수 있다. 하지만 필자라면 이걸 생각하기 위해 기꺼이 한 주중 하루를 고민할 것이다. 이게 차라리 SOAP 이니, WSDL이니 아니면 다른 요상한 기술들을 고민하는 것 보다 훨씬 낫기 때문이다.

보너스로 필자의 생각에는 모든 비즈니스 로직 인터페이스들을 REST 규칙에 맞게 따르는 것이 비즈니스 로직 자체를 훨씬 더 간결하고 깔끔하게 만들어준다고 본다. 오브젝트에 많은 액션이 담긴 컨트롤러들 만들어도 좋다. 하지만 REST 규칙을 통해 구성하면 멋지고 견고하게 구성할 수 있다. 마치 제약에서 벗어나는것 처럼 말이다.

아래 예제는 REST 규칙에 입각하여 route 맵을 만들었다. 각 컨트롤러에는 기본 CRUD 액션만 담겨있다.

```rb
resources :purchases, only: :create

resources :costs_calculations, only: :create

namespace :company do
  resource :account_details, only: :update
  resource :website_details, only: :update
  resource :contact_details, only: :update
end

namespace :balance do
  resources :funds, only: :create
end

resource :bank_account, only: :update
```

최상의 REST 설계를 하기위해 필자는 주로 아무런 기능을 구현하지 않은 상태에서 REST 액션에 리소스를 먼저 합쳐서 확인해본다. (예: `POST /balance/funds`) 이때, 작성한 route 네이밍이 마음에 들면 Rails route에 옮긴다. 사소하지만, Rails는 이러한 방식을 가장 잘 이해하고 따를수 있게 도와준다.

### 결론

Rails 컨트롤러를 분리하는 방법은 한 컨트롤러가 특정 기능만 담당하거나 너무 많은 로직들이 함께 있을때 또는 너무 많은 믹스인이 함께 있을때 매우 효과적이다.

그렇다고해서 추상화를 절대 하지마라는 의미는 아니다. 추상화 하는 작업은 이 다음에 생각해도 된다. 아마 특정 로직들은 몇몇의 컨트롤러에 공유되어 사용되어야 할 수도 있다. 가끔씩 하나의 액션만 가진 컨트롤러가 너무 비대해질수도 있다. 바로 이때, `concern`이나 모델 메소드, 아니면 백그라운드 잡을 사용 해야 할 시기인 것이다. 아니면 서비스 객체(너무 많이 사용하지는 말자.)를 사용해야 할 때일수도 있다.

어플리케이션이 커지면 커질수록 모든 기능을 이해하는데 더 오랜 시간이 걸릴수 있다. 이는 코드가 얼마나 깔끔한지와는 다른 문제이다. 그래도 컨트롤러를 분리하는 방법은 보는 사람을 좀 더 편안하게 만들어준다.


Original Source:
[How DHH Organizes His Rails Controllers](http://jeromedalbert.com/how-dhh-organizes-his-rails-controllers/)
