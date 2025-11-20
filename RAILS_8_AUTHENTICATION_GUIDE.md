# Rails 8 Authentication Guide - 2024/2025 Best Practices

## Overview

Rails 8's built-in auth generator provides a secure foundation. This guide covers verified 2024-2025 implementation best practices.

## Prerequisites

```bash
rails --version  # Requires Rails 8.0.0+
gem 'bcrypt', '~> 3.1.7'  # Add to Gemfile
```

## What's Generated vs What You Must Add

| Feature | Generated? | Notes |
|---------|-----------|-------|
| User/Session models, Login/Logout | ✅ Yes | `has_secure_password`, controllers, views |
| Password Reset, Rate Limiting | ✅ Yes | 15-min tokens, IP-based (10/3min) |
| Authentication Concern | ✅ Yes | `Current.user`, `start_new_session_for` |
| **Registration** | ❌ **NO** | Create RegistrationsController manually |
| **Validations** | ❌ **NO** | Add email/password validation yourself |
| **Account Lockout, 2FA, OAuth** | ❌ **NO** | Manual implementation required |

**Key Insight**: Rails 8 provides a foundation. You must add registration, validations, security features.

## Step 1: Generate & Migrate

```bash
bin/rails generate authentication && bin/rails db:migrate
```

Creates User/Session models, controllers, views, migrations.

## Step 2: Understanding Generated Models

**✅ User Model (Generated):**
```ruby
class User < ApplicationRecord
  has_secure_password  # BCrypt hashing, password/password_confirmation
  has_many :sessions, dependent: :destroy
  normalizes :email_address, with: ->(e) { e.strip.downcase }
end
```

**➕ Add Validations (Recommended):**
```ruby
validates :email_address, presence: true, uniqueness: true
validates :password, length: { minimum: 12 }, allow_nil: true
```

**✅ Session Model (Generated):**
```ruby
class Session < ApplicationRecord
  belongs_to :user
  # Token auto-generated via migration's token:token type (has_secure_token)
  # Tracks ip_address and user_agent for security
end
```

## Step 3: Add User Registration (Required)

**⚠️ Not Generated - Must Create Manually:**

```ruby
# app/controllers/registrations_controller.rb
class RegistrationsController < ApplicationController
  allow_unauthenticated_access only: [:new, :create]

  def new; @user = User.new; end

  def create
    @user = User.new(user_params)
    if @user.save
      start_new_session_for @user  # Helper from Authentication concern
      redirect_to root_path, notice: "Welcome!"
    else
      render :new, status: :unprocessable_entity
    end
  end

  private
  def user_params
    params.require(:user).permit(:email_address, :password, :password_confirmation)
  end
end
```

**Routes:** `resource :registration, only: [:new, :create]`

**View:** Standard form_with for email_address, password, password_confirmation fields.

## Step 4: Strengthen Security (Production)

**Password Complexity:**
```ruby
# app/models/user.rb
validates :password, length: { minimum: 12 },
  format: { with: /\A(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/,
            message: "needs upper/lower/number" }, allow_nil: true
```

**Session Config (config/initializers/session_store.rb):**
```ruby
Rails.application.config.session_store :cookie_store,
  expire_after: 12.hours, secure: Rails.env.production?,
  httponly: true, same_site: :lax
```

**Force SSL (config/environments/production.rb):**
```ruby
config.force_ssl = true
```

## Step 5: Optional Enhancements

**Account Lockout:**
```bash
rails g migration AddLockableToUsers failed_attempts:integer locked_at:datetime
# Then add to User: def locked?; locked_at.present? && locked_at > 1.hour.ago; end
```

**Email Verification:**
```bash
rails g migration AddConfirmationToUsers email_confirmed_at:datetime
```

## Critical Misconceptions

1. **Registration is included** → NO. Only login/logout generated.
2. **Password/email validations exist** → NO. Add manually.
3. **Sessions auto-expire** → NO. Configure timeout yourself.

Generator creates login + password reset. Everything else is YOUR responsibility.

## Password Reset (Included)

```ruby
user.password_reset_token  # 15-min signed token, no DB storage
User.find_by_password_reset_token(token)  # Verify & find user
```

## Production Checklist

- [ ] Force SSL + secure session cookies configured
- [ ] Email/password validations added to User model
- [ ] Registration controller created & tested
- [ ] Rate limiting verified, CSRF protection enabled
- [ ] Optional: email verification, 2FA, account lockout

## Common Pitfalls

1. Assuming validations exist (they don't - add manually)
2. Skipping email uniqueness validation (allows duplicate accounts)
3. Not configuring session expiration (sessions last forever)

## Resources

- [Rails Security Guide](https://guides.rubyonrails.org/security.html)
- [Rails 8 Auth Generator](https://blog.saeloun.com/2025/05/12/rails-8-adds-built-in-authentication-generator/)
- [OWASP Auth Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)

---

**Last Updated**: November 2025 | Rails 8.0+ | Verified against generator source code
