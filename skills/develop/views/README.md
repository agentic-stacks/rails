# Views: ERB, Layouts, Partials, and Helpers

Rails views are ERB templates rendered by controllers. This file covers the essentials for building full-stack Rails UIs.

## ERB Basics

ERB (Embedded Ruby) mixes Ruby into HTML using three tag types:

```erb
<%= expression %>   <%# outputs the result — HTML-escaped by default %>
<% statement %>     <%# executes Ruby, outputs nothing %>
<%# comment %>      <%# not rendered or executed %>
```

Output is HTML-escaped automatically. To output raw HTML (use with care):

```erb
<%= raw @article.body %>
<%# or %>
<%= @article.body.html_safe %>
```

### Instance Variables

Controllers assign instance variables that views can read:

```ruby
# app/controllers/articles_controller.rb
def show
  @article = Article.find(params[:id])
end
```

```erb
<%# app/views/articles/show.html.erb %>
<h1><%= @article.title %></h1>
<p><%= @article.body %></p>
```

---

## Layouts

### Default Layout

Every response is wrapped in `app/views/layouts/application.html.erb`. The controller's rendered view is inserted at `yield`:

```erb
<%# app/views/layouts/application.html.erb %>
<!DOCTYPE html>
<html>
  <head>
    <title><%= content_for?(:title) ? yield(:title) : "MyApp" %></title>
    <%= csrf_meta_tags %>
    <%= csp_meta_tag %>
    <%= stylesheet_link_tag "application" %>
    <%= javascript_importmap_tags %>
  </head>
  <body>
    <%= render "shared/navbar" %>
    <main>
      <%= yield %>
    </main>
    <%= render "shared/flash" %>
  </body>
</html>
```

### content_for and yield

Inject content from a view into a named slot in the layout:

```erb
<%# app/views/articles/show.html.erb %>
<% content_for :title, @article.title %>
<% content_for :sidebar do %>
  <nav>...</nav>
<% end %>

<h1><%= @article.title %></h1>
```

```erb
<%# app/views/layouts/application.html.erb %>
<title><%= yield(:title) || "MyApp" %></title>
<aside><%= yield(:sidebar) %></aside>
```

### Per-Controller Layouts

Rails looks for `app/views/layouts/<controller_name>.html.erb` automatically. Override explicitly:

```ruby
class AdminController < ApplicationController
  layout "admin"                        # always use admin layout
end

class SessionsController < ApplicationController
  layout "minimal", only: [:new]        # layout for specific actions
  layout false, only: [:destroy]        # no layout
end
```

### Nested Layouts

Render one layout inside another using `content_for` + `render template:`:

```erb
<%# app/views/layouts/admin.html.erb %>
<% content_for :body do %>
  <div class="admin-wrapper">
    <%= yield %>
  </div>
<% end %>
<%= render template: "layouts/application" %>
```

---

## Partials

Partials are reusable view fragments. File names begin with an underscore but are referenced without it.

### Basic Render

```erb
<%# renders app/views/shared/_navbar.html.erb %>
<%= render "shared/navbar" %>

<%# renders app/views/articles/_article.html.erb %>
<%= render "article", article: @article %>
```

### Local Variables

Pass locals explicitly to avoid implicit variable leakage:

```erb
<%= render "form", article: @article, url: articles_path %>
```

```erb
<%# app/views/articles/_form.html.erb %>
<%= form_with model: article, url: url do |f| %>
  <%= f.text_field :title %>
  <%= f.submit %>
<% end %>
```

### Collection Rendering

Render a partial for each item in a collection (one DB query, efficient):

```erb
<%# renders _article.html.erb for each article, exposes `article` local %>
<%= render @articles %>

<%# explicit form: %>
<%= render partial: "article", collection: @articles, as: :item %>
```

Add a spacer between items:

```erb
<%= render partial: "article", collection: @articles,
           spacer_template: "article_divider" %>
```

### Sharing Instance Variables vs. Locals

Partials can access controller instance variables, but prefer locals for reusability and testability. Use `local_assigns` to detect whether an optional local was passed:

```erb
<% title = local_assigns.fetch(:title, "Default Title") %>
```

---

## Built-in View Helpers

### Linking

```erb
<%= link_to "Home", root_path %>
<%= link_to "Article", article_path(@article) %>
<%= link_to "Delete", article_path(@article), data: { turbo_method: :delete,
                                                       turbo_confirm: "Sure?" } %>
<%= link_to "External", "https://example.com", target: "_blank", rel: "noopener" %>
```

### Forms

```erb
<%# model-backed form — method and URL inferred from object state %>
<%= form_with model: @article do |f| %>
  <%= f.label :title %>
  <%= f.text_field :title, placeholder: "Title", required: true %>

  <%= f.label :body %>
  <%= f.text_area :body, rows: 8 %>

  <%= f.check_box :published %>
  <%= f.label :published, "Publish now" %>

  <%= f.select :category, Article::CATEGORIES, include_blank: "Select..." %>
  <%= f.collection_select :author_id, @authors, :id, :name %>

  <%= f.submit "Save Article" %>
<% end %>

<%# URL-based (non-model) form %>
<%= form_with url: search_path, method: :get do |f| %>
  <%= f.text_field :q, placeholder: "Search..." %>
  <%= f.submit "Search" %>
<% end %>
```

### Images and Assets

```erb
<%= image_tag "logo.png", alt: "Logo", width: 200 %>
<%= image_tag @user.avatar, class: "avatar" %>   <%# absolute URL %>
<%= stylesheet_link_tag "application" %>
<%= javascript_importmap_tags %>
```

### Other Common Helpers

```erb
<%= number_to_currency(19.99) %>              <%# => $19.99 %>
<%= number_to_human(1_500_000) %>             <%# => 1.5 Million %>
<%= time_ago_in_words(@article.created_at) %> <%# => about 3 hours ago %>
<%= truncate(@article.body, length: 100) %>
<%= pluralize(@articles.count, "article") %>  <%# => "3 articles" %>
<%= simple_format(@article.body) %>           <%# wraps newlines in <p> tags %>
```

---

## Custom Helpers

Place helpers in `app/helpers/`. They are available to all views by default (Rails 7+).

```ruby
# app/helpers/articles_helper.rb
module ArticlesHelper
  def status_badge(article)
    css = article.published? ? "badge-green" : "badge-gray"
    content_tag(:span, article.status.capitalize, class: "badge #{css}")
  end

  def formatted_date(date, format: :long)
    return "N/A" unless date
    l(date, format: format)   # l() is the Rails localize helper
  end
end
```

```erb
<%= status_badge(@article) %>
<%= formatted_date(@article.published_at) %>
```

### Application Helper

Helpers in `app/helpers/application_helper.rb` are available everywhere:

```ruby
module ApplicationHelper
  def page_title(title = nil)
    base = "MyApp"
    title ? "#{title} | #{base}" : base
  end

  def flash_class(level)
    { notice: "alert-info", alert: "alert-danger" }.fetch(level.to_sym, "alert-secondary")
  end
end
```

---

## ViewComponent (Alternative to Partials)

[ViewComponent](https://viewcomponent.org) is a gem that replaces complex partials with Ruby objects. Use it when a partial needs logic, takes many arguments, or needs unit testing.

```bash
bundle add view_component
bin/rails generate component Article::Card title:string published:boolean
```

```ruby
# app/components/article/card_component.rb
class Article::CardComponent < ViewComponent::Base
  def initialize(article:)
    @article = article
  end

  def published_label
    @article.published? ? "Published" : "Draft"
  end
end
```

```erb
<%# app/components/article/card_component.html.erb %>
<div class="card">
  <h2><%= @article.title %></h2>
  <span class="label"><%= published_label %></span>
</div>
```

```erb
<%# in any view %>
<%= render Article::CardComponent.new(article: @article) %>
<%= render Article::CardComponent.with_collection(@articles, as: :article) %>
```

---

## Related Skills

- `hotwire.md` — Turbo Drive, Frames, Streams for SPA-like navigation
- `stimulus.md` — JavaScript sprinkles without a framework
- `action-cable.md` — real-time WebSocket updates
- `../assets/README.md` — choosing an asset pipeline
