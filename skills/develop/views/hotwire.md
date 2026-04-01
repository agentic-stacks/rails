# Hotwire: Turbo Drive, Frames, Streams, and Morphing

Hotwire (HTML Over The Wire) ships with Rails 7+ and provides SPA-like speed without writing JavaScript. It consists of Turbo and Stimulus. This file covers Turbo.

## Turbo Drive

Turbo Drive intercepts link clicks and form submissions, fetches the response via `fetch()`, and replaces `<body>` with the new content — keeping `<head>` in place. No full page reloads.

**It is automatic.** No opt-in required. Every link and form in a Turbo-enabled app goes through Turbo Drive by default.

### Disable Turbo Drive

```erb
<%# disable on a single link %>
<%= link_to "Download PDF", report_path(@report, format: :pdf), data: { turbo: false } %>

<%# disable on a form %>
<%= form_with model: @article, data: { turbo: false } do |f| %>

<%# disable for an entire section %>
<div data-turbo="false">
  <a href="/legacy-page">Legacy link</a>
</div>
```

Disable globally (rare — defeats the purpose):

```javascript
// app/javascript/application.js
import { Turbo } from "@hotwired/turbo-rails"
Turbo.session.drive = false
```

### Page Visit Types

- **Advance** (default): pushes a new history entry.
- **Replace**: replaces current history entry. Use `data-turbo-action="replace"` on a link.
- **Restore**: automatic on browser back/forward.

### Caching

Turbo Drive caches pages. On revisit it shows the cache immediately (preview), then fetches fresh content. Control caching:

```erb
<%# opt a page out of caching %>
<meta name="turbo-cache-control" content="no-cache">

<%# mark a page as requiring a full reload (e.g. after sign-in) %>
<meta name="turbo-cache-control" content="no-store">
```

### Progress Bar

A CSS progress bar appears for requests taking more than 500ms. Customize in CSS:

```css
.turbo-progress-bar {
  height: 3px;
  background-color: #0ea5e9;
}
```

---

## Turbo Frames

A Turbo Frame is a scoped region of the page. Navigation within a frame updates only that frame, not the full page.

```erb
<%# app/views/articles/index.html.erb %>
<%= turbo_frame_tag "article-list" do %>
  <%= render @articles %>
  <%= link_to "Next page", articles_path(page: @page + 1) %>
<% end %>
```

Any link inside a `turbo-frame` targets that frame by default. The response must contain a matching frame:

```erb
<%# app/views/articles/index.html.erb (response) %>
<%= turbo_frame_tag "article-list" do %>
  <%# updated content %>
<% end %>
```

### Target a Specific Frame from Outside

```erb
<%= link_to "Load preview", article_path(@article), data: { turbo_frame: "preview" } %>

<%= turbo_frame_tag "preview" do %>
  <%# updated here %>
<% end %>
```

### Break Out of a Frame

```erb
<%# navigate the whole page from inside a frame %>
<%= link_to "Full page", article_path(@article), data: { turbo_frame: "_top" } %>
```

### Lazy Loading

A frame with a `src` attribute loads its content asynchronously after the page renders:

```erb
<%= turbo_frame_tag "user-stats", src: user_stats_path(@user), loading: :lazy do %>
  <p>Loading...</p>
<% end %>
```

### Inline Editing Pattern

```erb
<%# show.html.erb %>
<%= turbo_frame_tag dom_id(@article) do %>
  <h1><%= @article.title %></h1>
  <%= link_to "Edit", edit_article_path(@article) %>
<% end %>

<%# edit.html.erb %>
<%= turbo_frame_tag dom_id(@article) do %>
  <%= form_with model: @article do |f| %>
    <%= f.text_field :title %>
    <%= f.submit "Save" %>
    <%= link_to "Cancel", article_path(@article) %>
  <% end %>
<% end %>
```

The edit form and cancel link both stay within the frame, so the rest of the page never changes.

---

## Turbo Streams

Turbo Streams deliver targeted DOM mutations from the server. They can be sent via:

- Form responses (HTML with MIME type `text/vnd.turbo-stream.html`)
- WebSocket / Action Cable
- Server-Sent Events

### Actions

| Action | Effect |
|---|---|
| `append` | Add content inside target, at the end |
| `prepend` | Add content inside target, at the start |
| `replace` | Replace the target element itself |
| `update` | Replace the content inside the target |
| `remove` | Remove the target element |
| `before` | Insert content before the target element |
| `after` | Insert content after the target element |
| `refresh` | Trigger a page refresh (Turbo 8+) |

### Turbo Stream Templates

Create a template named with `.turbo_stream.erb` and Rails responds with it automatically when the request accepts `text/vnd.turbo-stream.html`:

```erb
<%# app/views/articles/create.turbo_stream.erb %>
<%= turbo_stream.prepend "articles", partial: "article", locals: { article: @article } %>
<%= turbo_stream.update "article-count", @articles.count %>
<%= turbo_stream.replace "flash", partial: "shared/flash" %>
```

The controller needs no special changes — Rails picks the format automatically:

```ruby
# app/controllers/articles_controller.rb
def create
  @article = Article.new(article_params)
  if @article.save
    respond_to do |format|
      format.turbo_stream   # renders create.turbo_stream.erb
      format.html { redirect_to @article }
    end
  else
    render :new, status: :unprocessable_entity
  end
end
```

### Inline Stream Responses

Return streams without a template file:

```ruby
def create
  @article = Article.new(article_params)
  @article.save!
  render turbo_stream: [
    turbo_stream.prepend("articles", partial: "article", locals: { article: @article }),
    turbo_stream.update("flash", partial: "shared/flash")
  ]
end
```

### Form Validation (Unprocessable Entity)

Return `422 Unprocessable Entity` to re-render the form inside its frame:

```ruby
def create
  @article = Article.new(article_params)
  unless @article.save
    render :new, status: :unprocessable_entity   # Turbo replaces the frame
  end
end
```

---

## Turbo Stream Over Action Cable

Broadcast DOM updates from anywhere in your app — models, jobs, controllers:

```ruby
# app/models/article.rb
class Article < ApplicationRecord
  after_create_commit -> {
    broadcast_prepend_to "articles",
      partial: "articles/article",
      locals: { article: self },
      target: "articles"
  }

  after_update_commit -> { broadcast_replace_to "articles" }
  after_destroy_commit -> { broadcast_remove_to "articles" }

  # Shorthand for all three:
  broadcasts_to ->(article) { "articles" }
end
```

Subscribe in the view:

```erb
<%# app/views/articles/index.html.erb %>
<%= turbo_stream_from "articles" %>

<div id="articles">
  <%= render @articles %>
</div>
```

---

## Morphing (Rails 8 / Turbo 8)

Turbo 8 introduces page morphing: instead of replacing `<body>`, it diffs the DOM and applies only the changes. This preserves scroll position, focus, and ephemeral state (open dropdowns, etc.).

### Enable Morphing

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  # Enable morphing for all page refreshes
end
```

```erb
<%# app/views/layouts/application.html.erb %>
<%# Add to <head>: %>
<%= turbo_refreshes_with method: :morph, scroll: :preserve %>
```

Or per-view:

```erb
<%= turbo_refreshes_with method: :morph %>
```

### Trigger a Morph Refresh

```ruby
# Broadcast a page refresh (triggers morphing if enabled)
Turbo::StreamsChannel.broadcast_refresh_to "articles"

# From a model
after_update_commit -> { broadcast_refresh_to "articles" }
```

### Exclude Elements from Morphing

Mark elements you want to keep untouched during a morph:

```erb
<div data-turbo-permanent id="sidebar">
  <%# This element is never replaced during morphing %>
</div>
```

---

## Common Patterns

### Toast Notifications

```erb
<%# app/views/shared/_flash.html.erb %>
<div id="flash">
  <% flash.each do |type, message| %>
    <div class="alert alert-<%= type %>" data-controller="flash"
         data-flash-timeout-value="3000">
      <%= message %>
    </div>
  <% end %>
</div>
```

Stream a flash update after any action:

```erb
<%# included in most .turbo_stream.erb files %>
<%= turbo_stream.update "flash", partial: "shared/flash" %>
```

### Live Search

```erb
<%= form_with url: search_articles_path, method: :get,
              data: { controller: "search", action: "input->search#submit" } do |f| %>
  <%= f.text_field :q, data: { search_target: "input" } %>
<% end %>

<%= turbo_frame_tag "search-results" do %>
  <%= render @articles %>
<% end %>
```

```erb
<%# app/views/articles/search.turbo_stream.erb %>
<%= turbo_stream.update "search-results" do %>
  <%= render @articles %>
<% end %>
```

### Optimistic UI with Turbo Streams

Immediately append a placeholder, then replace it when the server responds:

```erb
<%# create.turbo_stream.erb %>
<%= turbo_stream.replace "new-article-placeholder",
      partial: "article", locals: { article: @article } %>
```

---

## Related Skills

- `stimulus.md` — add JavaScript behavior to Turbo-powered views
- `action-cable.md` — Action Cable channels, Solid Cable
- `README.md` — ERB, layouts, partials
