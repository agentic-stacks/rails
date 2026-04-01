# Security

Rails ships with broad, layered security protections. Understand what is on by default, what requires explicit configuration, and where your app is still exposed.

---

## Rails Built-in Protections

### CSRF Protection

Every non-GET request must carry a valid authenticity token. Rails verifies it automatically.

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  protect_from_forgery with: :exception  # default for HTML apps
end
```

Forms rendered with Rails helpers include the token automatically. For API-only apps, disable it and use token authentication instead:

```ruby
protect_from_forgery with: :null_session  # does not raise; clears session
# or skip entirely in API controllers:
skip_before_action :verify_authenticity_token
```

### SQL Injection Prevention

Active Record uses parameterized queries for all finder methods. Never interpolate user input into SQL strings.

```ruby
# Safe — parameterized
User.where("email = ?", params[:email])
User.where(email: params[:email])

# Unsafe — never do this
User.where("email = '#{params[:email]}'")  # SQL injection vulnerability
```

### XSS Auto-escaping

ERB templates escape all output by default. Every `<%= %>` call passes the value through `ERB::Util.html_escape`.

```erb
<%= @user.name %>        <!-- escaped: safe -->
<%= raw @user.bio %>     <!-- NOT escaped: only use for trusted content -->
<%= @user.bio.html_safe %> <!-- same risk as raw -->
```

### X-Frame-Options

Rails sets `X-Frame-Options: SAMEORIGIN` on every response, blocking clickjacking via iframes from external origins. Controlled via:

```ruby
# config/application.rb
config.action_dispatch.default_headers = {
  "X-Frame-Options" => "SAMEORIGIN"
}
```

### Strong Parameters

Controllers must explicitly whitelist every attribute before mass assignment reaches the model.

```ruby
def article_params
  params.require(:article).permit(:title, :body, :published)
  # Never use: params.require(:article).permit!
end
```

---

## OWASP Top 10 Mapping

| OWASP Risk | Rails Protection | Where to Read |
|---|---|---|
| A01 Broken Access Control | Pundit / CanCanCan policies, `before_action :authenticate_user!` | `authorization.md` |
| A02 Cryptographic Failures | `has_secure_password` (bcrypt), `force_ssl`, encrypted credentials | `authentication.md`, `hardening.md` |
| A03 Injection | Parameterized queries, never interpolate SQL | `hardening.md` |
| A04 Insecure Design | Policy objects, service layer design, threat modelling | `authorization.md` |
| A05 Security Misconfiguration | `brakeman`, `bundle-audit`, CSP headers, `config/credentials.yml.enc` | `hardening.md` |
| A06 Vulnerable Components | `bundle audit`, Dependabot, keep gems up to date | `hardening.md` |
| A07 Auth and Session Failures | `has_secure_password`, Devise, session expiry, token rotation | `authentication.md` |
| A08 Software/Data Integrity | `bundle audit --update`, signed/encrypted cookies | `hardening.md` |
| A09 Logging and Monitoring | `config.log_level`, `lograge`, filter_parameters | `../logging/` |
| A10 Server-Side Request Forgery | Validate and restrict outbound URLs; `ssrf_filter` gem | `hardening.md` |

---

## Files in This Skill

| File | What It Covers |
|---|---|
| `authentication.md` | Rails 8 auth generator, `has_secure_password`, Devise, JWT |
| `authorization.md` | Pundit, CanCanCan, role-based patterns, testing policies |
| `hardening.md` | CSP, CORS, rate limiting, SQL injection, XSS, CSRF, brakeman |

---

## Quick Audit Commands

```bash
# Static analysis — scans for known vulnerability patterns
bundle exec brakeman

# Dependency vulnerability scan
bundle exec bundle-audit check --update

# Check for outdated gems
bundle outdated --strict

# View currently set security headers (development)
curl -I http://localhost:3000 | grep -i "x-\|content-security\|strict"
```
