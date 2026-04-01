# Action Cable: WebSockets in Rails

Action Cable integrates WebSockets into Rails. It handles the server-side channel logic and the JavaScript subscription. For most Rails 7/8 apps, use Turbo Streams over Action Cable rather than the low-level API directly.

---

## Concepts

| Term | Meaning |
|---|---|
| **Connection** | One WebSocket connection per browser tab. Authentication lives here. |
| **Channel** | A logical stream, like a chat room or notifications feed. Multiple channels per connection. |
| **Subscription** | A client subscribing to a channel. |
| **Broadcasting** | Pushing data from the server to all subscribers of a stream. |
| **Stream** | A named pub/sub topic. A channel can have many streams. |

---

## Configuration

### cable.yml

```yaml
# config/cable.yml
development:
  adapter: async       # In-process, no external dependency. Development only.

test:
  adapter: test

production:
  adapter: redis
  url: <%= ENV.fetch("REDIS_URL") { "redis://localhost:6379/1" } %>
  channel_prefix: myapp_production
```

### Solid Cable (Rails 8 default)

Rails 8 ships Solid Cable, a database-backed adapter. No Redis required for small-to-medium traffic:

```yaml
# config/cable.yml (Rails 8)
production:
  adapter: solid_cable
  connects_to:
    database:
      writing: cable
  polling_interval: 0.1.seconds
  message_retention: 1.day
```

```ruby
# config/database.yml — add a cable database
production:
  cable:
    <<: *default
    database: myapp_cable_production
```

```bash
bin/rails solid_cable:install:migrations
bin/rails db:migrate
```

Switch back to Redis if you need sub-100ms latency or very high broadcast volume.

---

## Generate a Channel

```bash
bin/rails generate channel Chat
# creates:
#   app/channels/chat_channel.rb
#   app/javascript/channels/chat_channel.js
```

---

## Server-Side Channel

```ruby
# app/channels/chat_channel.rb
class ChatChannel < ApplicationCable::Channel
  def subscribed
    # Stream from a named topic
    stream_from "chat_#{params[:room]}"

    # Or stream from an Active Record model
    # stream_for current_user
  end

  def unsubscribed
    # Cleanup — called when client disconnects
    stop_all_streams
  end

  # Client can call channel.perform("speak", { message: "Hi" })
  def speak(data)
    Message.create!(body: data["message"], user: current_user)
  end
end
```

### Connection Authentication

```ruby
# app/channels/application_cable/connection.rb
module ApplicationCable
  class Connection < ActionCable::Server::Base::Connection
    identified_by :current_user

    def connect
      self.current_user = find_verified_user
    end

    private

    def find_verified_user
      # Session cookie is available in cookies[]
      if (user_id = cookies.encrypted[:user_id])
        User.find_by(id: user_id) || reject_unauthorized_connection
      else
        reject_unauthorized_connection
      end
    end
  end
end
```

With Devise, cookies are not automatically set. Use a signed global ID or a short-lived token instead:

```ruby
def find_verified_user
  verified_user = env["warden"].user
  verified_user || reject_unauthorized_connection
end
```

---

## Broadcasting

### From Anywhere

```ruby
# Broadcast a hash — client receives it as JSON
ActionCable.server.broadcast("chat_general", { message: "Hello", user: "Alice" })

# From a controller
def create
  @message = Message.create!(message_params)
  ActionCable.server.broadcast(
    "chat_#{@message.room_id}",
    { id: @message.id, body: @message.body, user: @message.user.name }
  )
  head :ok
end
```

### From a Model Callback

```ruby
class Message < ApplicationRecord
  belongs_to :room
  belongs_to :user

  after_create_commit :broadcast_message

  private

  def broadcast_message
    ActionCable.server.broadcast(
      "chat_#{room_id}",
      { body: body, user: user.name, created_at: created_at.strftime("%H:%M") }
    )
  end
end
```

---

## Client-Side Subscription

```javascript
// app/javascript/channels/chat_channel.js
import consumer from "./consumer"

const chatChannel = consumer.subscriptions.create(
  { channel: "ChatChannel", room: "general" },
  {
    connected() {
      console.log("Connected to ChatChannel")
    },

    disconnected() {
      console.log("Disconnected")
    },

    received(data) {
      // Append the message to the DOM
      const list = document.getElementById("messages")
      if (list) {
        list.insertAdjacentHTML("beforeend",
          `<div class="message"><strong>${data.user}:</strong> ${data.body}</div>`
        )
      }
    },

    // Call a server-side action
    speak(message) {
      this.perform("speak", { message })
    }
  }
)

// Call from a form submit handler:
// chatChannel.speak("Hello everyone!")
```

---

## Turbo Streams over Action Cable (Recommended)

For most use cases, skip the low-level channel API and use Turbo Streams. The model handles broadcasting and the view subscribes — no JavaScript to write.

### Model

```ruby
class Article < ApplicationRecord
  # Broadcasts Turbo Stream fragments to all subscribers
  broadcasts_to ->(article) { "articles" }

  # Or fine-grained control:
  after_create_commit  -> { broadcast_prepend_to "articles" }
  after_update_commit  -> { broadcast_replace_to "articles" }
  after_destroy_commit -> { broadcast_remove_to "articles" }
end
```

Available broadcast methods:

```ruby
broadcast_append_to   "stream"
broadcast_prepend_to  "stream"
broadcast_replace_to  "stream"
broadcast_update_to   "stream"
broadcast_remove_to   "stream"
broadcast_before_to   "stream"
broadcast_after_to    "stream"
broadcast_refresh_to  "stream"   # morphing page refresh (Turbo 8)

# With custom partial and locals:
broadcast_append_to "articles",
  target: "article-list",
  partial: "articles/article_card",
  locals: { article: self, highlight: true }
```

### View

```erb
<%# app/views/articles/index.html.erb %>
<%= turbo_stream_from "articles" %>

<div id="articles">
  <%= render @articles %>
</div>
```

`turbo_stream_from` renders a `<turbo-cable-stream-source>` element that opens a WebSocket subscription automatically.

### Scoped Streams (per user, per record)

```ruby
# Broadcast only to the article's author
after_update_commit -> {
  broadcast_replace_to [user, "articles"]
}
```

```erb
<%# Subscribe to the scoped stream %>
<%= turbo_stream_from [current_user, "articles"] %>
```

---

## Testing Action Cable

```ruby
# test/channels/chat_channel_test.rb
require "test_helper"

class ChatChannelTest < ActionCable::Channel::TestCase
  def setup
    stub_connection current_user: users(:alice)
  end

  test "subscribes and streams for room" do
    subscribe room: "general"
    assert subscription.confirmed?
    assert_has_stream "chat_general"
  end

  test "broadcasts message on speak" do
    subscribe room: "general"
    assert_broadcasts("chat_general", 1) do
      perform :speak, message: "Hello"
    end
  end
end
```

---

## Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| WebSocket not connecting | Missing `action_cable_meta_tag` | Add `<%= action_cable_meta_tag %>` to `<head>` |
| Connection rejected | Auth fails in `connect` | Check `find_verified_user` — ensure session cookie is set |
| Messages not received | Wrong stream name | Log stream names in `subscribed` and broadcast side |
| Redis errors in prod | Wrong URL or Redis down | Check `REDIS_URL` env var; consider Solid Cable |
| High memory in dev | Too many async threads | Normal — async adapter spins threads per connection |

---

## Related Skills

- `hotwire.md` — Turbo Streams, the recommended way to use Action Cable
- `stimulus.md` — JavaScript for user interactions
