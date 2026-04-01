# Stimulus: JavaScript Controllers for Rails

Stimulus is a modest JavaScript framework that connects HTML attributes to JavaScript objects. It lives alongside Turbo in the Hotwire stack. Controllers handle behavior; Turbo handles navigation and DOM updates.

## How Stimulus Works

1. Stimulus scans the DOM for `data-controller` attributes.
2. It instantiates the matching controller class.
3. The controller connects to its element and can observe targets, values, and actions.

---

## Controller Anatomy

```javascript
// app/javascript/controllers/hello_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  // Declare targets — elements the controller can reference
  static targets = ["output", "input"]

  // Declare values — typed data from data attributes
  static values = {
    url: String,
    delay: { type: Number, default: 300 },
    enabled: { type: Boolean, default: true }
  }

  // Declare CSS classes — class names from data attributes
  static classes = ["loading", "active"]

  // Lifecycle: called when the controller connects to the DOM
  connect() {
    console.log("Hello from", this.element)
  }

  // Lifecycle: called when the controller disconnects
  disconnect() {
    clearTimeout(this.timer)
  }

  // Action handler
  greet() {
    this.outputTarget.textContent = `Hello, ${this.inputTarget.value}!`
  }
}
```

### Registration

Stimulus auto-loads controllers using the `stimulus-loading` import map integration. File `hello_controller.js` maps to identifier `hello`. No manual registration needed in a standard Rails app.

For manual registration:

```javascript
// app/javascript/controllers/index.js
import { application } from "./application"
import HelloController from "./hello_controller"
application.register("hello", HelloController)
```

---

## HTML Attributes

### data-controller

Connects one or more controllers to an element. The element becomes the controller's `this.element`:

```erb
<div data-controller="hello">
  ...
</div>

<%# Multiple controllers on one element %>
<div data-controller="hello dropdown">
  ...
</div>
```

### data-action

Binds a DOM event to a controller action:

```erb
<%# Syntax: "event->controller#method" %>
<button data-action="click->hello#greet">Say Hello</button>

<%# Default event shorthand (omit event for common elements) %>
<%# button → click, input/select/textarea → input, form → submit %>
<button data-action="hello#greet">Say Hello</button>
<input data-action="hello#search">
<form data-action="hello#submit">

<%# Multiple actions %>
<input data-action="input->hello#search keydown.enter->hello#submit">

<%# Keyboard modifiers %>
<input data-action="keydown.enter->hello#submit">
<input data-action="keydown.escape->hello#cancel">
```

### data-*-target

Marks elements as targets the controller can reference:

```erb
<div data-controller="hello">
  <input data-hello-target="input" type="text">
  <span data-hello-target="output"></span>
</div>
```

In the controller:

```javascript
// this.inputTarget   → first matching element (throws if none)
// this.outputTarget  → first matching element
// this.inputTargets  → array of all matching elements
// this.hasInputTarget → boolean
```

### data-*-value

Pass typed data from HTML to the controller:

```erb
<div data-controller="autocomplete"
     data-autocomplete-url-value="/articles/search"
     data-autocomplete-delay-value="300"
     data-autocomplete-min-length-value="2">
</div>
```

```javascript
static values = {
  url: String,
  delay: { type: Number, default: 300 },
  minLength: { type: Number, default: 1 }   // camelCase maps to kebab-case attribute
}

connect() {
  console.log(this.urlValue)        // "/articles/search"
  console.log(this.delayValue)      // 300
}
```

Value types: `String`, `Number`, `Boolean`, `Array`, `Object`.

### data-*-class

Externalize CSS class names so controllers stay style-agnostic:

```erb
<div data-controller="spinner"
     data-spinner-loading-class="opacity-50 cursor-wait"
     data-spinner-active-class="ring ring-blue-500">
</div>
```

```javascript
static classes = ["loading", "active"]

start() {
  this.element.classList.add(...this.loadingClasses)
}

stop() {
  this.element.classList.remove(...this.loadingClasses)
}
```

---

## Lifecycle Callbacks

```javascript
export default class extends Controller {
  // Called once when the controller is first instantiated
  initialize() { }

  // Called each time the controller connects to the DOM (including after Turbo navigation)
  connect() { }

  // Called when the controller disconnects
  disconnect() { }

  // Called when a target is added to the DOM
  inputTargetConnected(element) { }

  // Called when a target is removed from the DOM
  inputTargetDisconnected(element) { }

  // Called when a value changes (including on connect)
  urlValueChanged(value, previousValue) { }
}
```

---

## Common Patterns

### Toggle

```javascript
// app/javascript/controllers/toggle_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["content"]
  static classes = ["hidden"]
  static values = { open: { type: Boolean, default: false } }

  toggle() {
    this.openValue = !this.openValue
  }

  openValueChanged(open) {
    this.contentTarget.classList.toggle(this.hiddenClass || "hidden", !open)
  }
}
```

```erb
<div data-controller="toggle">
  <button data-action="toggle#toggle">Toggle</button>
  <div data-toggle-target="content" class="hidden">
    Hidden content
  </div>
</div>
```

### Debounced Search

```javascript
// app/javascript/controllers/search_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["input"]
  static values = { delay: { type: Number, default: 300 } }

  connect() {
    this.timer = null
  }

  disconnect() {
    clearTimeout(this.timer)
  }

  search() {
    clearTimeout(this.timer)
    this.timer = setTimeout(() => {
      this.element.requestSubmit()
    }, this.delayValue)
  }
}
```

```erb
<%= form_with url: search_path, method: :get,
              data: { controller: "search", turbo_frame: "results" } do |f| %>
  <%= f.text_field :q,
        data: { action: "input->search#search", search_target: "input" },
        autocomplete: "off" %>
<% end %>

<%= turbo_frame_tag "results" do %>
  <%= render @articles %>
<% end %>
```

### Form Validation

```javascript
// app/javascript/controllers/form_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["submit"]

  connect() {
    this.validate()
  }

  validate() {
    const valid = this.element.checkValidity()
    this.submitTarget.disabled = !valid
  }
}
```

```erb
<%= form_with model: @article, data: { controller: "form" } do |f| %>
  <%= f.text_field :title, required: true,
        data: { action: "input->form#validate" } %>
  <%= f.submit "Save", data: { form_target: "submit" } %>
<% end %>
```

### Clipboard Copy

```javascript
// app/javascript/controllers/clipboard_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["source", "feedback"]

  copy() {
    navigator.clipboard.writeText(this.sourceTarget.value).then(() => {
      this.feedbackTarget.textContent = "Copied!"
      setTimeout(() => { this.feedbackTarget.textContent = "" }, 2000)
    })
  }
}
```

```erb
<div data-controller="clipboard">
  <input data-clipboard-target="source" value="<%= @api_key %>" readonly>
  <button data-action="clipboard#copy">Copy</button>
  <span data-clipboard-target="feedback"></span>
</div>
```

### Modal

```javascript
// app/javascript/controllers/modal_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["container"]

  open() {
    this.containerTarget.classList.remove("hidden")
    document.body.classList.add("overflow-hidden")
  }

  close(event) {
    if (event.target === this.containerTarget || event.key === "Escape") {
      this.containerTarget.classList.add("hidden")
      document.body.classList.remove("overflow-hidden")
    }
  }

  // Close on backdrop click
  backdropClose(event) {
    if (event.target === event.currentTarget) this.close(event)
  }
}
```

---

## Using Third-Party Stimulus Libraries

Popular libraries extend Stimulus with common controllers:

```bash
# stimulus-use: composable behaviors (debounce, observe, click-outside, etc.)
bin/importmap pin stimulus-use

# stimulus-components: pre-built controllers
bin/importmap pin @stimulus-components/clipboard
```

```javascript
import { useDebounce } from "stimulus-use"

export default class extends Controller {
  static debounces = ["search"]

  connect() {
    useDebounce(this, { wait: 300 })
  }

  search() {
    // debounced automatically
    this.element.requestSubmit()
  }
}
```

---

## Generating Controllers

```bash
bin/rails generate stimulus hello
# creates app/javascript/controllers/hello_controller.js
```

---

## Related Skills

- `hotwire.md` — Turbo Drive, Frames, Streams (what Stimulus complements)
- `action-cable.md` — WebSockets for real-time updates
