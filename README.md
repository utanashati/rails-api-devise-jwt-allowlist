# Rails 7 API-only Setup with Devise & Devise JWT

I'm using rbenv as my ruby env manager and rbenv-gemset extension to create collections of gems.

To set this up:

### Create a new API-only Rails project

```bash
brew upgrade ruby-build  # On MacOS to update the Ruby versions list
rbenv install 3.3.4
rbenv global 3.3.4

gem install rails

rails new my_app --api --database=postgresql --skip-test  # Don't forget to add your preferred options
```

### Setup CORS

1. Uncomment `gem "rack-cors"` in `Gemfile`

2. `bundle install`

3. Uncomment CORS config in `cors.rb`. For dev purposes, let any connection through:

```rb
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins "*"

    resource "*",
      headers: :any,
      methods: [ :get, :post, :put, :patch, :delete, :options, :head ]
  end
end
```

### Setup [Devise](https://github.com/heartcombo/devise)

1. `bundle add devise`

2. `rails generate devise:install`

3. Add the following line to `development.rb`:

```rb
config.action_mailer.default_url_options = { host: 'localhost', port: 3000 }
```

4. `rails generate devise User`

5. Uncomment the "Confirmable" section in the `devise_create_users.rb` migration.

6. `rails db:create db:migrate`

7. Follow these:

   1. https://github.com/waiting-for-dev/devise-jwt/wiki/Configuring-devise-for-APIs#responding-to-json
   2. https://github.com/waiting-for-dev/devise-jwt/wiki/Configuring-devise-for-APIs#defaulting-to-json-as-format

8. Add `devise :confirmable` to the `User` model.

9. Use [letter_opener](https://github.com/ryanb/letter_opener) to open confirmation letters in development (follow the link for instructions).

### Setup [Devise JWT](https://github.com/waiting-for-dev/devise-jwt)

1. `bundle add devise-jwt`

2. Generate a secret: `rails secret`

3. `EDITOR="code --wait" rails credentials:edit`:

```yml
devise_jwt_secret_key: <your_secret>
```

**Note!** Devise JWT [encourage](https://github.com/waiting-for-dev/devise-jwt?tab=readme-ov-file#secret-key-configuration) to use a secret different from the `secret_key_base`.

4. Add the config to `devise.rb`:

```rb
Devise.setup do |config|
  # ...
  config.jwt do |jwt|
    jwt.secret = Rails.application.credentials.devise_jwt_secret_key!
  end
end
```

5. Create `allowlisted_jwts` table:

`rails g migration create_allowlisted_jwts`:

```rb
def change
  create_table :allowlisted_jwts do |t|
    t.string :jti, null: false
    t.string :aud, null: false
    t.datetime :exp, null: false
    t.references :user, foreign_key: { on_delete: :cascade }, null: false  # user singular (!)
  end

  add_index :allowlisted_jwts, :jti, unique: true
end
```

`rails db:migrate`

6. Create `AllowlistedJwt` model:

```rb
class AllowlistedJwt < ApplicationRecord
end
```

7. Include JWT in `User`:

```rb
class User < ApplicationRecord
  include Devise::JWT::RevocationStrategies::Allowlist

  devise ...
         :jwt_authenticatable, jwt_revocation_strategy: self
end
```

8. Add the `aud_header` config to `devise.rb`:

```rb
Devise.setup do |config|
  # ...
  config.jwt do |jwt|
    jwt.secret = Rails.application.credentials.devise_jwt_secret_key!
    jwt.aud_header = "JWT_AUD"  # Change to the preferred one
  end
end
```

**This header needs to be in your request headers**, or else there's going to be an error:

`ActiveRecord::NotNullViolation (PG::NotNullViolation: ERROR:  null value in column "aud" of relation "allowlisted_jwts" violates not-null constraint)`

## Testing with Postman

`rails s`

![POST /users](docs/post_users.png)

![Headers](docs/headers.png)
