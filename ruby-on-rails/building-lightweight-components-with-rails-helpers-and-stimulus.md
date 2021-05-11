## Tip: Building lightweight components with Rails Helpers and Stimulus

Custom Rails `helpers` modules are often overlooked, but they can be a great option for building lightweight components and reducing boilerplate in your Stimulus controllers.

One nice thing about Stimulus is that you can quickly infer the functionality just from reading the markup attributes, but for components that have a couple of values and actions, you can benefit from hiding some of the implementation details.

Let’s take the example from my [GitHub-style Hovercards article](https://boringrails.com/articles/hovercards-stimulus/):

```erb
<div class="inline-block"
  data-controller="hovercard"
  data-hovercard-url-value="<%= hovercard_shoe_path(shoe) %>"
  data-action="mouseenter->hovercard#show mouseleave->hovercard#hide"
>
  <%= link_to shoe.name, shoe, class: "branded-link" %>
</div>
```
The `hovercard_controller` needs to be passed in a `url` value and have two actions added for showing and hiding the card on hover. The controller is wrapped around a link that can be styled and customized for each type of hovercard we want in the app.

If you have only a few places using the controller, it’s not a big deal. But if you want to [re-use this controller](https://boringrails.com/articles/better-stimulus-controllers) for more and more types of hovercards, try adding your own Rails helper.

Usage
Modules in the `app/helpers` folder will automatically be available for you to use in your views.

```rb
# app/helpers/hovercard_helper.rb
module HovercardHelper

  # Use a helper to avoid repeating Stimulus controller attributes
  def hovercard(url, &block)
    content_tag(:div,
      "data-controller": "hovercard",
      "data-hovercard-url-value": url,
      "data-action": "mouseenter->hovercard#show mouseleave->hovercard#hide",
      &block)
  end

  # Build your own light-weight "components"
  def repo_hovercard(repo, &block)
    hovercard hovercard_repository_path(repo), &block
  end

  def user_hovercard(user, &block)
    hovercard hovercard_user_path(user), &block
  end
end
```

Using a helper allows us to create our own “components” in Ruby to abstract away the implementation details. And because of the power of [Ruby blocks](https://www.codewithjason.com/understanding-ruby-blocks), we can create flexible components that can be customized per usage.

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
For example, we could build a repo_hovercard helper that accepts a Repository model and a block to render. We have full control over what the display based on the page context but we don’t want to worry about wiring up Stimulus events correct.

And if we want to change our Stimulus controller, it’s all in one spot, instead of spread out across many views in the app.

Additional Resources
Rails API: Helpers

Original Source:
[Tip: Building lightweight components with Rails Helpers and Stimulus](https://boringrails.com/tips/lightweight-components-with-helpers-stimulus)
