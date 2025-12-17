# Authentication Patterns

> Passwordless magic links without Devise - ~150 lines of custom code.

---

## Why Not Devise?

Devise is powerful but heavyweight. For passwordless auth, custom code is simpler:
- No password storage complexity
- No password reset flows
- Fewer dependencies
- Full control over the flow

## Magic Link Flow

```
1. User enters email
2. Server generates signed token, emails link
3. User clicks link
4. Server validates token, creates session
```

## Identity Model

Separate global identity from per-account users:

```ruby
class Identity < ApplicationRecord
  has_many :users
  has_many :sessions

  normalizes :email, with: ->(email) { email.strip.downcase }

  generates_token_for :magic_link, expires_in: 15.minutes
  generates_token_for :email_verification, expires_in: 24.hours
end
```

## Session Model

```ruby
class Session < ApplicationRecord
  belongs_to :identity
  belongs_to :user, optional: true

  before_create :set_user_agent_and_ip

  def self.start!(identity:, user_agent:, ip_address:)
    identity.sessions.create!(
      user_agent: user_agent,
      ip_address: ip_address
    )
  end

  private
    def set_user_agent_and_ip
      self.user_agent = Current.user_agent
      self.ip_address = Current.ip_address
    end
end
```

## Magic Link Controller

```ruby
class Sessions::MagicLinksController < ApplicationController
  rate_limit to: 10, within: 15.minutes, only: :create

  def new
  end

  def create
    if identity = Identity.find_by(email: params[:email])
      SessionMailer.magic_link(identity).deliver_later
    end

    # Always show success (don't reveal if email exists)
    redirect_to root_path, notice: "Check your email for a login link"
  end

  def show
    identity = Identity.find_by_token_for(:magic_link, params[:token])

    if identity
      session = Session.start!(identity: identity)
      set_session_cookie(session)
      redirect_to after_login_path
    else
      redirect_to new_session_path, alert: "Invalid or expired link"
    end
  end
end
```

## Session Cookie

```ruby
module Authentication
  extend ActiveSupport::Concern

  included do
    before_action :set_current_session
    helper_method :signed_in?, :current_user
  end

  private
    def set_current_session
      if session_token = cookies.signed[:session_token]
        Current.session = Session.find_by(token: session_token)
        Current.user = Current.session&.user
      end
    end

    def set_session_cookie(session)
      cookies.signed.permanent[:session_token] = {
        value: session.signed_id,
        httponly: true,
        same_site: :lax,
        path: Current.account&.slug || "/"
      }
    end

    def signed_in?
      Current.session.present?
    end

    def current_user
      Current.user
    end

    def require_authentication
      redirect_to new_session_path unless signed_in?
    end
end
```

## Magic Link Email

```ruby
class SessionMailer < ApplicationMailer
  def magic_link(identity)
    @identity = identity
    @url = session_magic_link_url(token: identity.generate_token_for(:magic_link))

    mail to: identity.email, subject: "Your login link"
  end
end
```

```erb
<%# app/views/session_mailer/magic_link.html.erb %>
<p>Click the link below to sign in:</p>

<p><%= link_to "Sign in", @url %></p>

<p>This link expires in 15 minutes.</p>
```

## Current Context

```ruby
class Current < ActiveSupport::CurrentAttributes
  attribute :session, :user, :identity, :account
  attribute :user_agent, :ip_address

  def user=(user)
    super
    self.identity = user&.identity
    self.account = user&.account
  end
end
```

## Multi-Account Support

Users can belong to multiple accounts via the same identity:

```ruby
class User < ApplicationRecord
  belongs_to :identity
  belongs_to :account

  # Same person, different accounts
  # identity.users.count > 1
end
```

## Session Path Scoping

For multi-tenant apps, scope session cookie to account path:

```ruby
cookies.signed.permanent[:session_token] = {
  value: session.signed_id,
  path: "/#{account.external_id}"  # e.g., "/1234567"
}
```

**Why**: Allows simultaneous login to multiple accounts without cookie conflicts.

## Key Principles

1. **Passwordless is simpler** - No password storage, reset flows, or breach liability
2. **Rate limit aggressively** - Prevent email bombing
3. **Never reveal email existence** - Always show "check your email"
4. **Signed tokens with expiry** - Rails 7.1+ `generates_token_for`
5. **Separate identity from user** - One person, many accounts
