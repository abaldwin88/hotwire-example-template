# Hotwire: Server-rendered live previews

[![Deploy to Heroku](https://www.herokucdn.com/deploy/button.png)][heroku-deploy-app]

[heroku-deploy-app]: https://heroku.com/deploy?template=https://github.com/thoughtbot/hotwire-example-template/tree/hotwire-example-live-preview

The upcoming [Rails 7][] release will be notable for numerous reasons.
One of the more remarkable being the deprecation and removal of Rails
UJS, and the introduction of [Hotwire][] as Rails' [default JavaScript
framework][].

Let's learn more about how Hotwire fits into Rails by building an
Article drafting experience that provides end-users with a preview of
their final version as they type!

We'll start with an out-of-the-box Rails installation that utilizes
Turbo Drive, Turbo Streams, and Stimulus to then progressively enhance
concepts and tools that are built directly into browsers. Plus, it'll
degrade gracefully when JavaScript is unavailable!

The code samples contained within omit the majority of the application's
setup. While reading, know that the bulk of the application's code was
generated by a `rails new` command

The rest of the source code from this article can be found [on
GitHub][].

[Hotwire]: https://hotwired.dev
[Rails 7]: https://edgeguides.rubyonrails.org/7_0_release_notes.html
[default JavaScript framework]: https://github.com/rails/rails/pull/42999
[on GitHub]: https://github.com/thoughtbot/hotwire-example-template/commits/hotwire-example-live-preview

## Drafting Articles

We'll start by using Rails `model` generator to create some `Article`
[scaffolding][] code to serve as a starting point for our extensions and
customizations:

```sh
bin/rails generate scaffold Article content:text
bin/rails db:migrate
```

We won't be making any changes to the vast majority of the generated code, but
there are two view partials that will be changed the most throughout the rest of
the article. To have context for those changes, it's worth becoming familiar
with them before we get started: `app/views/articles/_form.html.erb` and
`app/views/articles/_article.html.erb`.

The `articles/form` partial renders a single `<textarea>` field for our
`Article` model's `content`, along with an `<input type="submit">`
element:

```erb
<%= form_with(model: article) do |form| %>
  <% if article.errors.any? %>
    <div id="error_explanation">
      <h2><%= pluralize(article.errors.count, "error") %> prohibited this article from being saved:</h2>

      <ul>
        <% article.errors.each do |error| %>
          <li><%= error.full_message %></li>
        <% end %>
      </ul>
    </div>
  <% end %>

  <div class="field">
    <%= form.label :content %>
    <%= form.text_area :content %>
  </div>

  <div class="actions">
    <%= form.submit %>
  </div>
<% end %>
```

Similarly, the `articles/article` partial renders the value of the
`content` as plain text:

```erb
<div id="<%= dom_id article %>" class="scaffold_record">
  <p>
    <strong>Content:</strong>
    <%= article.content %>
  </p>

  <p>
    <%= link_to "Show this article", article %>
  </p>
</div>
```

The rest of the generated code serves as the foundation for creating,
reading, updating, and destroying `Article` instances. While it's
crucial, they're implementation details that we can ignore for the sake
of this example.

[scaffolding]: https://edgeguides.rubyonrails.org/getting_started.html#mvc-and-you-generating-a-model

## Previewing our changes

<abbr title="Hyper-text transfer protocol">HTTP</abbr> request-response
exchanges and full <abbr title="Hyper-text markup language">HTML</abbr>
documents will serve as a stable foundation for the feature.

Let's implement it through `<form>` submissions that are handled by Rails'
router, controller, and view layers. The HTTP response will redirect
_back_ to the `Article` form, with the contents encoded into the URL as
a query parameter, and rendered as a preview of a published `Article`.

In this first version, the application doesn't make use of **any**
Hotwire concepts.

First, let's introduce a `:create` route that handles `POST` requests to
generate `Article` previews:

```diff
--- a/config/routes.rb
+++ b/config/routes.rb
 Rails.application.routes.draw do
   resources :articles
+  resources :previews, only: :create
   # For details on the DSL available within this file, see https://guides.rubyonrails.org/routing.html
 end
```

Next, we'll declare a `<button>` element within the `article/form`
partial's `<form>` element to submit to the new `POST /previews` route:

```diff
--- a/app/views/articles/_form.html.erb
+++ b/app/views/articles/_form.html.erb
   <div class="actions">
     <%= form.submit %>
+    <%= form.button "Preview Article", formaction: previews_path %>
   </div>
 <% end %>
```

The `<button>` element affords an alternative means of submitting the
form by declaring a [formaction="/previews"][button-formaction]
attribute. Setting `[formaction="/previews"]` overrides _where_ the
`<form>` submits to when the `<button>` is clicked, enabling the
secondary `<button>` to supplement the primary submission `<button>`,
which continues to submit to the route specified by the `<form
action="/articles">` element's `[action]` attribute when clicked.

To ensure that clicking the `<button>` submits a `POST` request,
regardless of the `<form method="…">` attribute or a [Rails-generated
`<input type="hidden" name="_method" value="…">` element][_method],
invoke `form.method` with `name:` and `value:` options:

```diff
--- a/app/views/articles/_form.html.erb
+++ b/app/views/articles/_form.html.erb
   <div class="actions">
     <%= form.submit %>
-    <%= form.button "Preview Article", formaction: previews_path %>
+    <%= form.button "Preview Article", formaction: previews_path,
+          name: "_method", value: "post" %>
   </div>
 <% end %>
```

The `PreviewsController` declared in
`app/controllers/previews_controller.rb` handles the `POST /previews`
request by constructing an instance of an `Article` from `params`
corresponding to the `<form>` element's submitted fields, then redirects
_back_ to the `GET /articles/new` route:

```ruby
class PreviewsController < ApplicationController
  def create
    @preview = Article.new(article_params)

    redirect_to new_article_url(article: @preview.attributes)
  end

  private

  def article_params
    params.require(:article).permit(:content)
  end
end
```

When redirecting to the `GET /articles/new` route, we'll encode the
`content` value into URL query parameters by passing the `Hash` returned
by [Article#attributes][attributes] to the `new_article_url` under the
`article` scope.

It's worth noting that  according to the [HTTP specification][], there
are no limits on the length of a URI:

> The HTTP protocol does not place any a priori limit on the length of
> a URI. Servers MUST be able to handle the URI of any resource they
> serve, and SHOULD be able to handle URIs of unbounded length if they
> provide GET-based forms that could generate such URIs.
>
> - 3.2.1 General Syntax

In practice, [conventional wisdom][] suggests that URLs over 2,000
characters are risky.

[HTTP specification]: https://tools.ietf.org/html/rfc2616#section-3.2.1
[conventional wisdom]: https://stackoverflow.com/a/417184

To read those values and pass them along to the `Article` instance
constructed by the `articles#new` action, we'll need to make some tweaks
to the `ArticlesController#article_params` and `ArticlesController#new`
methods so that the `articles#new` action supports reading from query
parameters when they're present:

```diff
--- a/app/controllers/articles_controller.rb
+++ b/app/controllers/articles_controller.rb
@@ -12,7 +12,7 @@ class ArticlesController < ApplicationController

   # GET /articles/new
   def new
-    @article = Article.new
+    @article = Article.new(article_params)
   end

   # GET /articles/1/edit
@@ -53,6 +53,6 @@ class ArticlesController < ApplicationController

     # Only allow a list of trusted parameters through.
     def article_params
-      params.require(:article).permit(:content)
+      params.fetch(:article, {}).permit(:content)
     end
 end
```

The `Article` record assigned to the `@article` instance variable is
eventually passed to the `articles/form` partial and made available
through the `article` partial-local variable. Within the `articles/form`
partial, render the `articles/article` partial using the pre-populated
`article` reference:

```diff
--- a/app/views/articles/_form.html.erb
+++ b/app/views/articles/_form.html.erb
   </div>

+  <div class="field">
+    <strong>Preview:</strong>
+    <div id="article_preview">
+      <%= render partial: "articles/article", object: article %>
+    </div>
+  </div>
+
   <div class="actions">
 <% end %>
```

To ensure a consistent structure between the `articles#show` and the
`articles/content` partial rendered within the `articles#new` template,
we'll change the `articles/article` partial to also render the
`articles/content` partial.

```diff
--- a/app/views/articles/_article.html.erb
+++ b/app/views/articles/_article.html.erb
 <div id="<%= dom_id article %>" class="scaffold_record">
   <p>
     <strong>Content:</strong>
-    <%= article.content %>
+    <%= render partial: "articles/content", object: article.content %>
   </p>
```

The `app/views/articles/_content.html.erb` partial will serve as an
example of server-side rendering. In our case, we'll transform the
`content` into HTML by passing its plain-text value to Action View's
[simple_format][] helper:

```html+erb
<%= simple_format content %>
```

The `#simple_format` view helper wraps any text separate by two newlines
(`\n\n`) in `<p>` elements.

We're being intentional minimal here. If you're feeling imaginative,
consider an `articles/content` partial with more a complicated set of
transformations (like Markdown parsing or other rich-text styling).

[button-formaction]: https://developer.mozilla.org/en-US/docs/Web/HTML/Element/button#attr-formaction
[_method]: https://edgeguides.rubyonrails.org/form_helpers.html#how-do-forms-with-patch-put-or-delete-methods-work-questionmark
[attributes]: https://edgeapi.rubyonrails.org/classes/ActiveRecord/AttributeMethods.html#method-i-attributes
[simple_format]: https://edgeapi.rubyonrails.org/classes/ActionView/Helpers/TextHelper.html#method-i-simple_format

## Progressively Enhancing the experience with Hotwire

Since Rails generators provide out-of-the-box integration with [Turbo
and Stimulus][hotwire-generators], our application's `<a>`-initiated
navigations and `<form>`-powered submissions are augmented by [Turbo
Drive][].

In spite of that, we haven't capitalized on any opportunities to
implement any [Hotwire][]-specific capabilities. Let's build atop the
foundation we've established for `POST /previews` requests by
progressively enhancing the experience with [Turbo Stream][].

After we make our changes, responses to `POST /previews` submissions
will embed server-generated HTML into [`<turbo-stream>`][Turbo Stream]
elements, which our client-side application can render.

In this case, Turbo will process the `<turbo-stream action="replace">`
elements that contain the Article preview HTML by finding corresponding
elements within the client's current document replacing their HTML
contents, without redirecting or navigating the page, all without a
single line of application-specific JavaScript.

[Turbo Stream]: https://turbo.hotwired.dev/handbook/streams
[hotwire-generators]: https://github.com/rails/rails/blob/d5b9618da1ac60c674e8a27c3ab4e33742e9aa9b/railties/lib/rails/generators/app_base.rb#L327-L331

Turbo Streaming updates to the document
---

On the client-side, [Turbo Drive][] monitors our page's `submit` events,
and intervenes whenever `<form>` elements are  submitted. During
`<form>` submissions, Turbo Drive will ensure that the resulting HTTP
requests are transmitted with the [Accept: text/vnd.turbo-stream.html,
text/html, application/xhtml+xml][Accept] header.

On the server-side, [Turbo Rails][] handles requests with `Accept:
text/vnd.turbo-stream.html, text/html, application/xhtml+xml` by
categorizing them with the `turbo_stream` format in the same way that
Rails transforms `Accept: text/html` requests into an `html` format or
`Accept: application/json` requests into a `json` format.

Having a dedicated `turbo_stream` format affords Turbo Rails
applications with the full suite of Rails' rendering of capabilities.
For example, we can leverage the dedicated `turbo_stream` rendering
format by modifying our controller code to render the
`previews/create.turbo_stream.erb` template when responding to requests
with the Turbo Stream [Accept][] header:

```diff
--- a/app/controllers/previews_controller.rb
+++ b/app/controllers/previews_controller.rb
   def create
     @preview = Article.new(article_params)

-    redirect_to new_article_url(article: @preview.attributes)
+    respond_to do |format|
+      format.html { redirect_to new_article_url(article: @preview.attributes) }
+      format.turbo_stream
+    end
   end
```

The template for the `articles#create` action (declared in
`app/views/previews/create.turbo_stream.erb`) embeds our
`articles/article` partial into a `<turbo-stream>` element:

```html+erb
<%= turbo_stream.update "article_preview" do %>
  <%= render partial: "articles/article", object: @preview, as: :article %>
<% end %>
```

We're using the [Turbo Rails]-provided `turbo_stream` helper to generate
our `<turbo-stream>` elements. The [update][] action is most appropriate
for our use case, since it will replace the descendants of the targeted
element while retaining the element itself.

Since we're hard-coding the `<turbo-stream>` element's [target][]
attribute to `article_preview`, we're coupling our template to the
requesting page's structure and naming. If we were to pass the
`article_preview` along in the request, we could make our response more
flexible _and_ limit the appearance of that value to a single place: the
`article/form` partial. We can pass the identifier value along by
encoding it into the URL as the `render_into` query parameter:

```diff
--- a/app/views/articles/_form.html.erb
+++ b/app/views/articles/_form.html.erb
   <div class="actions">
     <%= form.submit %>
     <%= form.button "Preview Article", name: "_method", value: "post",
-          formaction: previews_path %>
+          formaction: previews_path(render_into: "article_preview") %>
   </div>
 <% end %>
```

Then, we'll change our response template to read the `"article_preview"`
value from the `params[:render_into]` key:

```diff
- <%= turbo_stream.update "article_preview" do %>
+ <%= turbo_stream.update params[:render_into] do %>
    <%= render partial: "articles/article", object: @preview, as: :article %>
  <% end %>
```

Let's review!
---

At this point, our end-users can draft an `Article` and can preview
their changes by clicking on a `Preview Article` button to request the
server-rendered HTML version.

[Hotwire]: https://hotwire.dev/
[Turbo Drive]: https://turbo.hotwire.dev/handbook/drive
[Accept]: https://turbo.hotwire.dev/handbook/streams#streaming-from-http-responses
[Turbo Rails]: https://github.com/hotwired/turbo-rails
[turbo-stream]: https://turbo.hotwire.dev/handbook/streams
[update]: https://turbo.hotwire.dev/reference/streams#update
[target]: https://turbo.hotwire.dev/handbook/streams#stream-messages-and-actions

## Live previews as you type

Let's enhance that experience even more by cutting out the need to click
the `Preview Article` button.

[Stimulus][] is one of packages included in the Hotwire suite. [Stimulus
Controllers][] enable our applications to transform [browser-side
events][] into function calls on our controller instances by routing
them as [Stimulus Actions][].

We'll start by declaring a `form` controller to augment our `<form>`
element. To utilize the `form` [identifier][], we'll declare the
`app/javascript/controllers/form_controller.js` file:

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
}
```

Then we'll render the `<form>` element with `[data-controller="form"]`
attribute:

```diff
--- a/app/views/articles/_form.html.erb
+++ b/app/views/articles/_form.html.erb
-<%= form_with(model: article) do |form| %>
+<%= form_with(model: article, data: { controller: "form" }) do |form| %>
    <% if article.errors.any? %>
      <div id="error_explanation">
        <h2><%= pluralize(article.errors.count, "error") %> prohibited this article from being saved:</h2>
```

To grant direct access to the `<button>` element's `HTMLButtonElement`
instance, we'll make our `form` controller aware of it by declaring
`[data-form-target="preview"]` attribute on the on the `<button>`:

```diff
--- a/app/views/articles/_form.html.erb
+++ b/app/views/articles/_form.html.erb
   <div class="actions">
     <%= form.submit %>
     <%= form.button "Preview Article", formaction: previews_path(render_into: "article_preview"),
-          name: "_method", value: "post" %>
+          name: "_method", value: "post",
+          data: { form_target: "preview" } %>
   </div>
 <% end %>
```

In our controller, we'll add support for accessing the
`HTMLButtonElement` instance through the `previewTarget` property by
declaring `"preview"` as a [Stimulus Target][]:

```diff
--- b/app/javascript/controllers/form_controller.js
+++ b/app/javascript/controllers/form_controller.js
  import { Controller } from "@hotwired/stimulus"

  export default class extends Controller {
+   static get targets() { return [ "preview" ] }
  }
```

Whenever the contents of `<input>`, `<select>`, or `<textarea>`
elements' values change, browsers fire an [input][] event. To route
those events as [Stimulus Actions][], we'll declare
`[data-action="input->form#preview"]` on the `<form>` element:

```diff
--- a/app/views/articles/_form.html.erb
+++ b/app/views/articles/_form.html.erb
-<%= form_with(model: article, data: { controller: "form" }) do |form| %>
+<%= form_with(model: article, data: { controller: "form", action: "input->form#preview" }) do |form| %>
    <% if article.errors.any? %>
      <div id="error_explanation">
        <h2><%= pluralize(article.errors.count, "error") %> prohibited this article from being saved:</h2>
```

Whenever any `<form>` element descendant fires an [input][] event,
Stimulus will respond by calling the `preview()` method in the `form`
controller (as instructed by the directive declared in the
`form[data-action]` attribute).

The `preview()` action handles the event by finding the element
annotated with the `[data-form-target="preview"]` attribute (in this
case, our `Preview Post` button), and programmatically clicking it.

```diff
--- b/app/javascript/controllers/form_controller.js
+++ b/app/javascript/controllers/form_controller.js
  import { Controller } from "@hotwired/stimulus"

  export default class extends Controller {
    static get targets() { return [ "preview" ] }
+
+   preview() {
+     this.previewTarget.click()
+   }
  }
```

When end-users visit the application with JavaScript-enabled browsers,
their changes to the `<form>` fields will submit the `<form>`
automatically.

In the spirit of progressive enhancement, we'll want to ensure that the
feature gracefully degrades when JavaScript is unavailable. To do so,
we'll make sure that the `Preview Post` is _always_ rendered to the
page. Then whenever JavaScript is available, we'll hide it. We'll extend
the `form` controller to manage the `<button>` element's visibility.

In the absence of JavaScript, Turbo won't have an opportunity to inject
the `Accept: text/vnd.turbo-stream.html` into the `Accept` header, so
requests made by submitting the `<form>` will have the `Accept:
text/html` header. When those requests are handled by our server, the
response will redirect like it did before we introduced any
`turbo_stream`-specific code.

During the `form` controller's lifecycle, Stimulus invokes the
[connect()][] function. Once the controller is connected, we can hide
the `<button>` by annotating it with the [hidden][] attribute:

```diff
  import { Controller } from "@hotwired/stimulus"

  export default class extends Controller {
   static get targets() { return [ "preview" ] }
+
+   connect() {
+     this.previewTarget.hidden = true
+   }

    preview() {
      this.previewTarget.click()
    }
  }
```

Improving even further
---

For example, if we want to guard against flooding our clients and
servers with extraneous requests whenever a swift keyboardist quickly
enters text, it would be worthwhile to [debounce][] preview submissions:

```diff
--- a/app/javascript/controllers/form_controller.js
+++ b/app/javascript/controllers/form_controller.js
@@ -1,8 +1,13 @@
 import { Controller } from "@hotwired/stimulus"
+import debounce from "https://cdn.skypack.dev/lodash.debounce"

 export default class extends Controller {
   static get targets() { return [ "preview" ] }

+  initialize() {
+    this.preview = debounce(this.preview.bind(this), 300)
+  }
+
   connect() {
     this.previewTarget.hidden = true
   }

   preview() {
     this.previewTarget.click()
   }
 }
```

Wrapping up
---

We've built an Article drafting experience that provides end-users with
a preview of the final version, live as they type. The experience relies
on Turbo Streams and Stimulus to progressively enhance tools and
concepts built directly into browsers, and will gracefully degrade to
relying upon HTML and HTTP in scenarios where JavaScript is unavailable.

Our implementation is light on JavaScript code, never encodes our
`Article` records into JSON representations, and doesn't include a
single line of application-specific code calling to [XMLHttpRequest][]
or [fetch][], in spite of sourcing all of its HTML's structure and data
from the server.

These omissions are at the core of Hotwire's value proposition. Turbo,
specifically, demonstrates that applications can treat `<form>` elements
as declarative HTML alternatives to imperative Asynchronous JavaScript
and XML (<abbr title="Asynchronous JavaScript and XML">AJAX</abbr>)
invocations. They sit within a page's document, inert and ready to be
executed at a moment's notice by end-users. An application's `<form>`
elements are the bedrock for any and all Hotwire-augmented end-user
experiences.

[browser-side events]: https://developer.mozilla.org/en-US/docs/Learn/JavaScript/Building_blocks/Events
[stimulus]: https://stimulus.hotwire.dev
[Stimulus Controllers]: https://stimulus.hotwire.dev/reference/controllers
[Stimulus Actions]: https://stimulus.hotwire.dev/reference/actions
[identifier]: https://stimulus.hotwire.dev/reference/controllers#identifiers
[Stimulus Target]: https://stimulus.hotwire.dev/reference/targets#properties
[connect()]: https://stimulus.hotwire.dev/reference/lifecycle-callbacks#connection
[input]: https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement/input_event
[hidden]: https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/hidden
[debounce]: https://docs-lodash.com/v4/debounce/
[XMLHttpRequest]: https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest
[fetch]: https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API
