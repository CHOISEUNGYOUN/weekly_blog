## How DHH Organizes His Rails Controllers

In a recent interview with Full Stack Radio our Lord and Savior DHH™ explains how he organizes Rails controllers in the latest version of Basecamp. Here is an enhanced transcript of his holy words:

What I’ve come to embrace is that being almost fundamentalistic about when I create a new controller to stay adherent to REST has served me better every single time. Every single time I’ve regretted the state of my controllers, it’s been because I’ve had too few of them. I’ve been trying to overload things too heavily.

So, in Basecamp 3 we spin off controllers every single time there’s even sort of a subresource that makes sense. Which can be things like filters. Like, say, you have this screen and it looks in a certain state. Well, if you apply a couple of filters and dropdowns to it, it’s in a different state. Sometimes we just take those states and we make a brand new controller for it.

The heuristics I use to drive that is: whenever I have the inclination that I want to add a method on a controller that’s not part of the default five or whatever REST actions that we have by default, make a new controller! And just call it that.

So let’s say you have an `InboxController` and you have an `index` that shows everything that’s in the inbox; and you might have another action where you go like “Oh I wanna see like the pending. Just show me the pending emails in that or something like that”. So you add an action called `pendings`:

```rb
class InboxesController < ApplicationController
  def index
  end

  def pendings
  end
end
```

That’s a very common pattern right? And a pattern that I used to follow more.

Now I just go like “no no no”. Have a new controller that’s called `Inboxes::PendingsController` that just has a single method called `index`:

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

And what I found is that the freedom that gives you is that each controller now has its own scope with its own sets of filters that apply […].

So we’ve had great controller proliferation and especially controller proliferation within namespaces. So let’s say we have a `MessagesController` and below that controller we might have a `Messages::DraftsController` and we might have a `Messages::TrashesController` and we might have all these other subcontrollers or subresources within the same thing. That’s been a huge success.

Amen.

So basically he says that controllers should only use the default CRUD actions `index`, `show`, `new`, `edit`, `create`, `update`, `destroy`. Any other action would lead to the creation of a dedicated controller (which itself only has default CRUD actions).

### What I think about it

From now on the following opinions are my own. Different opinions are totally OK. Just saying, before you call me a pontificating zealot. Calm down guys!

Anyways, I was happy to learn that I have been using “the DHH way” to organize controllers for more than a year now #DHHFanboy. The examples he mentions are only about filtering though, and are probably overkill for simple controller logic. A common way to filter in REST is by using query parameters (e.g. `GET /inboxes?state=pending`), so I would stick to that when the code is short and simple (once it gets long or complicated or there are too many mixed actions/concerns, I would do the same as him).

But I totally agree with the general idea of splitting controllers. I like it for several reasons.

### It encourages you to produce simpler code

With this technique you can create as many controllers as you want.
Use your judgment though: If the controller only has default CRUD actions and is relatively short/simple (like the ones scaffolded by Rails), there is probably no need to prematurely extract each `index`/`show`/etc into their own controller.

Where the controller splitting technique becomes very cool is when the controller is getting heavier, even when it has only default CRUD actions. What to do then? Well, just put the heavy code in its own dedicated controller!

For example, here is what our most complicated controller looks like at my current company (we are using “relatively skinny models and fat controllers” so YMMV). It allows you to purchase a product in the API part of our app:

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

There is only one public method (the default CRUD `create` action). No premature abstractions into fat models, no “service classes”, no observers, no nothing, no bullshit. Everything is here, nicely contained in the controller. No need to jump between a dozen files to understand what is going on; you just need to open this one file in your code editor. And since controllers are the entry point for any web application code, you would open that file anyway when coding. #ObviousCode

But you may ask “Jeez, how long is that class since you dump everything in it? Surely it is very long, like a gazillion lines, right?” Nope. 144 lines.

And that’s our most complicated controller. Like, the worst of the worst. We could arguably split its code into smaller chunks but it looks good enough to us (YMMV). The rest of our other “fat” controllers are much simpler: between 6 and 103 lines long, with an average of **15 lines of code** per controller (we have about 150 controllers for now).

Remember the projects you worked on where controllers were 200+ lines long AND this was only a small portion of the code related to the request — the rest of it being scattered between a gazillion of service objects, observers, and models? This kind of shit doesn’t happen here, mostly thanks to this controller splitting technique, among other easy things like the [rule of 3](https://en.wikipedia.org/wiki/Rule_of_three_(computer_programming)). Indeed, [duplication is far cheaper than the wrong abstraction](https://sandimetz.com/blog/2016/1/20/the-wrong-abstraction), and it’s one of the reasons why I think (like DHH) that [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) and [SRP](https://en.wikipedia.org/wiki/Single-responsibility_principle) are crap and they are overrated and they shouldn’t be an end in themselves and they suck balls and and and… Anywaaays, back to our topic!

### It makes your code more uniform

Knowing that there can only be so many CRUD actions in a controller is quite cool. No more guessing/spelunking/endlessly scrolling in long controllers to find that one weird action. No more wondering how/if a custom controller method maps to a route.

I don’t like to be [surprised](https://en.wikipedia.org/wiki/Principle_of_least_astonishment) when doing mundane organization. I like uniform code, and heavy [convention over configuration](https://rubyonrails.org/doctrine/#convention-over-configuration) is one of the many reasons why I still prefer Rails compared to other Ruby frameworks. Everything is organized the same, so you spend less time making mundane decisions, and make faster progress in areas that really matter for the business.

In theory, it also means that you can move from one codebase to another and be 100% productive in a very short time. In practice people seem to encounter many “horrible Rails apps” in the wild. One company may use architectural patterns such as [observers](https://github.com/rails/rails-observers) (not my cup of tea), another might use a whole additional architecture layer such as [Trailblazer](https://github.com/trailblazer/trailblazer) (not my cup of tea either but it has some interesting ideas), another company will use yet another tool, another one will use its own custom sauce, etc.

All of this because people seem uneasy and unhappy with the so-called “lack of structure” in vanilla Rails apps. So they look for additional structure elsewhere. Guys! The solution is right under your nose! Split your controllers, and only use the default CRUD actions. Simple as pie, and junior developer friendly.

Rails could possibly do a better job of promoting this controller splitting heuristics. Their [doc](https://guides.rubyonrails.org/routing.html#non-resourceful-routes) only briefly says “[…] you should usually use resourceful routing […]”. But the idea of CRUD actions and RESTful routes has [prominently been featured](https://guides.rubyonrails.org/routing.html#resource-routing-the-rails-default) in their doc for ages. If you RTFM’ed, it has probably crossed your mind at least once that adding custom actions (apart from the CRUD ones) is not very “Rails way”. Splitting controllers is a good answer to that uneasy feeling.

### It makes you think in terms of REST

A lot of people like REST because it is uniform and simple. Once you understand a (truly) RESTful API, it is easier to understand another one. In theory at least: authentication anyone? ;–). The business logic obviously differs between applications so you have to understand it, but how you consume that logic is the same: you [create a charge](https://stripe.com/docs/api#create_charge) in Stripe (i.e. you take someone’s money), you [create an SMS](https://www.twilio.com/docs/sms/api/message-resource) in Twilio (i.e. you send it), you [get a repository](https://docs.github.com/en/rest/reference/repos#get) in Github, etc.

You have to bend your mind a little at first to use REST nouns instead of actions: you don’t “pay”, you “create a payment”. You don’t “add funds to your balance”, you “create a fund in a balance”. Et cetera. Maybe a bit weird at first, but I would pay this small price any day of the week rather than going back to SOAP, WSDL and all that jazz (ex-Java/JEE developers will know what I’m talking about).

As a bonus, I think that having your whole business logic interface (not necessarily implementation) dictated by REST makes for a cleaner and simpler business logic. You can only have objects with so many actions: no more, no less. Yet you know you can express anything with REST, and it will be sound and dependable. It is a liberating constraint.

Here are some example RESTful Rails routes that map to splitted controllers that only use CRUD actions.

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
For best REST design (especially when subresources are involved), I usually write down the REST action+resource first on a temporary note (example: `POST /balance/funds`) without worrying about the implementation. Then when I am happy with the naming, I translate it to a Rails route, which is trivial since Rails has very good REST support.

### Wrapping up

Splitting your Rails controllers when they have a very specific scope, too much logic, or too many mixed concerns can have a lot of good side effects in your code.

It doesn’t mean that you never abstract. It just comes later down the road. At some point some logic needs to be shared by several controllers. Sometimes even a splitted controller with only one public method gets too big. Et cetera. This is where concerns, model methods, possibly background jobs, and even sometimes service objects (hopefully not too many) come into play.

The more your app grows, the more time you will need to spend to understand it, no matter how clean the code is. But splitting your controllers makes things easier.


Original Source:
[How DHH Organizes His Rails Controllers](http://jeromedalbert.com/how-dhh-organizes-his-rails-controllers/)
