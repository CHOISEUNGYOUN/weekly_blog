## Rails Helpers 모듈과 Stimulus를 활용한 가벼운 컴포넌트 만들기

우리는 종종 커스텀 Rails `helpers` 모듈은 간과하지만 잘 사용한다면 가벼운 단위의 컴포넌트 구성과 Stimulus 컨트롤러의 보일러플레이트 코드를 줄이는데 유용한 수단이 될 수 있다.

Stimulus의 장점은 마크업 요소를 읽기만해도 해당 컴포넌트가 어떤 기능을 하는지 빨리 추론할 수 있다는 것 뿐만 아니라 여러 변수와 동작을 요구하는 컴포넌트들의 동작 디테일을 추상화 시킬수 있다.

여기 [GitHub-style Hovercards article](https://boringrails.com/articles/hovercards-stimulus/) 언급했던 예시를 보자.

```erb
<div class="inline-block"
  data-controller="hovercard"
  data-hovercard-url-value="<%= hovercard_shoe_path(shoe) %>"
  data-action="mouseenter->hovercard#show mouseleave->hovercard#hide"
>
  <%= link_to shoe.name, shoe, class: "branded-link" %>
</div>
```

`hovercard_controller`는 `url` 변수를 받음과 동시에 마우스 커서를 올려놓을 시 카드를 보여주고 숨기는 동작을 추가한다. 컨트롤러는 각각의 hovercard 마다 css 커스터마이징이 가능한 link 태그를 감싸고 있다.

이 컨트롤러를 많이 사용하지 않는다면 문제 될일은 없으나, 다양한 종류의 hovercard에 사용을 하고 싶다면 [재사용성을 감안하여](https://boringrails.com/articles/better-stimulus-controllers) Rails helper에 추가를 해보자.

### 사용예시

`app/helpers` 폴더에 있는 모듈들은 자동으로 view에서 사용 가능하다.

```rb
# app/helpers/hovercard_helper.rb
module HovercardHelper

  # Stimulus 요소들을 반복 적용하는것을 피하기 위해 helper 모듈을 사용.
  def hovercard(url, &block)
    content_tag(:div,
      "data-controller": "hovercard",
      "data-hovercard-url-value": url,
      "data-action": "mouseenter->hovercard#show mouseleave->hovercard#hide",
      &block)
  end

  # 가벼운 컴포넌트를 만듬.
  def repo_hovercard(repo, &block)
    hovercard hovercard_repository_path(repo), &block
  end

  def user_hovercard(user, &block)
    hovercard hovercard_user_path(user), &block
  end
end
```

Helper를 사용하면 Ruby 내에서 동작 조건에 대한 세부 내용들을 추상화 할 수 있게 도와준다. 이는 [Ruby blocks](https://www.codewithjason.com/understanding-ruby-blocks) 의 강력한 기능이다. 이 기능을 활용하면 어느 상황에도 맞추어 적용할 수 있는 유연한 컴포넌트를 만들 수 있다.

```erb
<!-- app/views/timeline.html.erb -->

<%= user_hovercard(@user) do %>
  <%= link_to @user.username, @user %>
<% end %>

<%= repo_hovercard(repository) do %>
  <div class="flex items-center space-x-2">
    <svg></svg> <!-- Some icon -->
    <%= link_to repository.name, repository %>
  </div>
<% end %>
```

`Repository` 모델을 변수로 받고 블럭을 렌더 해주는 `repo_hovercard` helper 를 만드는 것 처럼 매번 helper 메소드를 만들 수도 있다. 이는 각 페이지 컨텍스트 마다 동작되는 `hovercard`를 하나의 helper 모듈에서 컨트롤 하는 장점 뿐만 아니라 매번 Stimulus 로 UI 이벤트를 구성하는 문제도 덜어준다.

그리고 Stimulus 컨트롤러를 수정하고 싶으면 하나의 컨트롤러를 수정하기만 하면 된다. 여러 view 페이지들을 일일이 확인해가며 수정할 필요가 없는 것이다.

추가 참고 사항
Rails API: [Helpers](https://api.rubyonrails.org/classes/ActionController/Helpers.html)

Original Source:
[Tip: Building lightweight components with Rails Helpers and Stimulus](https://boringrails.com/tips/lightweight-components-with-helpers-stimulus)
