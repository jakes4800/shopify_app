# Upgrading

This file documents important changes needed to upgrade your app's Shopify App version to a new major version.

#### Table of contents

[General Advice](#general-advice)

[Unreleased](#unreleased)

[Upgrading to `v22.2.0`](#upgrading-to-v2220)

[Upgrading to `v22.0.0`](#upgrading-to-v2200)

[Upgrading to `v20.3.0`](#upgrading-to-v2030)

[Upgrading to `v20.2.0`](#upgrading-to-v2020)

[Upgrading to `v20.1.0`](#upgrading-to-v2010)

[Upgrading to `v19.0.0`](#upgrading-to-v1900)

[Upgrading to `v18.1.2`](#upgrading-to-v1812)

[Upgrading to `v17.2.0`](#upgrading-to-v1720)

[Upgrading to `v13.0.0`](#upgrading-to-v1300)

[Upgrading to `v11.7.0`](#upgrading-to-v1170)

[Upgrading from `v8.6` to `v9.0.0`](#upgrading-from-v86-to-v900)

## General Advice

Although we strive to make upgrades as smooth as possible, some effort may be required to stay up to date with the latest changes to `shopify_app`.

We strongly recommend you avoid 'monkeypatching' any existing code from `ShopifyApp`, e.g. by inheriting from `ShopifyApp` and then overriding particular methods. This can result in difficult upgrades. If your app does so, you will need to carefully check the gem's internal changes when upgrading.

If you need to upgrade by more than one major version (e.g. from v18 to v20), we recommend doing one at a time. Deploy each into production to help to detect problems earlier.

We also recommend the use of a staging site which matches your production environment as closely as possible.

If you do run into issues, we recommend looking at our [debugging tips.](https://github.com/Shopify/shopify_app/blob/main/docs/Troubleshooting.md#debugging-tips)

## Unreleased

#### (v23.0.0) Drops support for Ruby 3.0
The minimum ruby version is now 3.1

#### (v23.0.0) - Deprecated methods in CallbackController
The following methods from `ShopifyApp::CallbackController` have been deprecated in `v23.0.0`
- `perform_after_authenticate_job`
- `install_webhooks`
- `perform_post_authenticate_jobs`

If you have overwritten these methods in your callback controller to modify the behavior of the inherited `CallbackController`, you will need to
update your app to use configurable option `config.custom_post_authenticate_tasks` instead. See [post authenticate tasks](/docs/shopify_app/authentication.md#post-authenticate-tasks)
for more information.

#### (v23.0.0) - Removed `ShopifyApp::JWTMiddleware`
The `ShopifyApp::JWTMiddleware` middleware has been removed in `v23.0.0`. This middleware was used to populate the following environment variables from the JWT session token:
- `request.env["jwt.token"]`
- `request.env["jwt.shopify_domain"]`
- `request.env["jwt.shopify_user_id"]`
- `request.env["jwt.expire_at"]`

If you are using any of these variables in your app, you'll need to replace them. You can instead include the `ShopifyApp::WithShopifyIdToken` concern, which does the same JWT parsing as the middleware, and exposes the same values in the following helper methods:
- `shopify_id_token`
- `jwt_shopify_domain`
- `jwt_shopify_user_id`
- `jwt_expire_at`

#### (v23.0.0) - Deprecated "ShopifyApp::JWT" class
The `ShopifyApp::JWT` class has been deprecated in `v23.0.0`. Use [ShopifyAPI::Auth::JwtPayload](https://github.com/Shopify/shopify-api-ruby/blob/main/lib/shopify_api/auth/jwt_payload.rb)
class from the `shopify_api` gem instead. A search and replace should be enough for this migration.
  - `ShopifyAPI::Auth::JwtPayload` is a superset of the `ShopifyApp::JWT` class, and contains methods that were available in `ShopifyApp::JWT`.
  - `ShopifyAPI::Auth::JwtPayload` raises `ShopifyAPI::Errors::InvalidJwtTokenError` if the token is invalid.

## Upgrading to `v22.2.0`
#### Added new feature for zero redirect embedded app authorization flow - Token Exchange
A new embedded app authorization strategy has been introduced in `v22.2.0` that eliminates the redirects that were previously necessary for OAuth.
It can replace the existing installation and authorization code grant flow.
See [new embedded app authorization strategy](/README.md#new-embedded-app-authorization-strategy-token-exchange) for more information.

## Upgrading to `v22.0.0`
#### Dropped support for Ruby 2.x
Support for Ruby 2.x has been dropped as it is no longer supported. You'll need to upgrade to 3.x.x

#### Renamed Controller Concerns
The following controller concerns have been renamed/replaced in `v21.10.0` and have now been removed. To upgrade, please rename any usage in your apps's controllers that include them to the following:

|Old Deprecated Controller Concern |Replaced By New Controller Concern|
|---|---|
|`Authenticated`|`EnsureHasSession`|
|`RequireKnownShop`|`EnsureInstalled`|

The new names better reflect what assurances the including the controller concern provide. The new concern provide similar if not identical functionality as the concerns they replaced.

#### Remove ScripttagManager
Script tag usage has largely been replaced with the adoption of [theme app extensions](https://shopify.dev/docs/apps/online-store/theme-app-extensions) and [thank you order status customization](https://shopify.dev/docs/apps/checkout/thank-you-order-status). The manager has been removed with this major release due to effective replacement and a goal to have parity in supported functionality across language stacks.

If you find yourself still using Scipt Tags and want to continue the pattern of declarative management of script tags this gem used to use, we recommend porting the logic [the manager used in prior versions](https://github.com/Shopify/shopify_app/blob/2336fabc6d0b45a4dee3f336455dace4d2d88bc4/lib/shopify_app/managers/scripttags_manager.rb#L4) and implementing it in a [post authentication job](https://github.com/Shopify/shopify_app/blob/main/docs/shopify_app/authentication.md#run-jobs-after-the-oauth-flow). This is the recommended flow to create script tags (or any other logic) for stores that install your app.

#### No longer rescue non-shopify API errors during customized OAuth flow
If you have customized authentication logic and are counting on the `CallbackController` to catch your error and redirect to login, you'll need to catch that error and redirect to `login_url_with_optional_shop`.

## Upgrading to 21.3.0
The `Itp` controller concern has been removed from `LoginProtection` which is included by the `Authenticated`/`EnsureHasSession` controller concern.
If any of your controllers are dependant on methods from `Itp` then you can include `ShopifyApp::Itp` directly.
You may notice a deprecation notice saying, `Itp will be removed in an upcoming version`.
This is because we intend on removing `Itp` completely in `v22.0.0`, but this will work in the meantime.

## Upgrading to `v20.3.0`
Calling `LoginProtection#current_shopify_domain` will no longer raise an error if there is no active session. It will now return a nil value. The internal behavior of raising an error on OAuth redirect is still in place, however. If you were calling `current_shopify_domain` in authenticated actions and expecting an error if nil, you'll need to do a presence check and raise that error within your app.

## Upgrading to `v20.2.0`

All custom errors defined inline within the `ShopifyApp` gem have been moved to `lib/shopify_app/errors.rb`.

- If you rescue any errors defined in this gem, you will need to rename them to match their new namespacing.

## Upgrading to `v20.1.0`

Note that the following steps are *optional* and only apply to **embedded** applications. However, they can improve the loading time of your embedded app at installation and re-auth.

- For embedded applications, update any controller that renders a full page reload (e.g: your home controller) to redirect using `Shopify::Auth.embedded_app_url`, if the `embedded` query argument is not present or does not equal `1`. Example [here](https://github.com/Shopify/shopify-app-template-ruby/pull/35/files#)
- If your app already has a frontend that uses App Bridge, this gem now supports using that to redirect out of the iframe before OAuth.  Example [here](https://github.com/Shopify/shopify-frontend-template-react/blob/main/pages/ExitIframe.jsx)
  - In your `shopify_app.rb` initializer, configure `.embedded_redirect_url` to the path of the route you added above.
  - If you don't set this route, then the `shopify_app` gem will automatically load its own copy of App Bridge and perform this redirection without any additional configuration.

## Upgrading to `v19.0.0`

There are several major changes in this release:

* A change of strategy regarding sessions: Due to security changes with browsers, support for cookie based sessions was dropped. JWT is now the only supported method for managing sessions.
* As part of that change, this update moves API authentication logic from this gem to the [`shopify_api`](https://github.com/Shopify/shopify-api-ruby) gem.
* Previously the `shopify_api` gem relied on `ActiveResource`, an outdated library which was [removed](https://github.com/rails/rails/commit/f1637bf2bb00490203503fbd943b73406e043d1d) from Rails in 2012. v10 of `shopify_api` has a replacement approach which aims to provide a similar syntax, but changes will be necessary.

### High-level process

- Delete `config/initializers/omniauth.rb` as apps no longer need to initialize `OmniAuth` directly.
- Delete `config/initializers/user_agent.rb` as `shopify_app` will set the right `User-Agent` header for interacting
  with the Shopify API. If the app requires further information in the `User-Agent` header beyond what Shopify API
  requires, specify this in the `ShopifyAPI::Context.user_agent_prefix` setting.
- Remove `allow_jwt_authentication=` and `allow_cookie_authentication=` invocations from
  `config/initializers/shopify_app.rb` as the decision logic for which authentication method to use is now handled
  internally by the `shopify_api` gem, using the `ShopifyAPI::Context.embedded_app` setting.
- [Follow the guidance for upgrading `shopify-api-ruby`](https://github.com/Shopify/shopify-api-ruby#breaking-change-notice-for-version-1000).

### Specific cases

#### Shopify user ID in session

Previously, we set the entire app user object in the `session` object.
As of v19, since we no longer save the app user to the session (but only the shopify user id), we now store it as `session[:shopify_user_id]`. Please make sure to update any references to that object.

#### Webhook Jobs

It is assumed that you have an ActiveJob implementation configured for `perform_later`, e.g. Sidekiq.
Ensure your jobs inherit from `ApplicationJob` or `ActiveJob::Base`.

Add a new `handle` method to existing webhook jobs to go through the updated `shopify_api` gem.

```ruby
class MyWebhookJob < ActiveJob::Base
  extend ShopifyAPI::Webhooks::Handler

  class << self
    # new handle function
    def handle(topic:, shop:, body:)
      # delegate to pre-existing perform_later function
      perform_later(topic: topic, shop_domain: shop, webhook: body)
    end
  end

  # original perform function
  def perform(topic:, shop_domain:, webhook:)
    # ...
```

#### Temporary sessions

The new `shopify_api` gem offers a utility to temporarily create sessions for interacting with the API within a block.
This is useful for interacting with the Shopify API outside of the context of a subclass of `AuthenticatedController`.

```ruby
ShopifyAPI::Auth::Session.temp(shop: shop_domain, access_token: shop_token) do |session|
  # make invocations to the API
end
```

Within a subclass of `AuthenticatedController`, the `current_shopify_session` function will return the current active
Shopify API session, or `nil` if no such session is available.

#### Setting up `ShopifyAPI::Context`

The `shopify_app` initializer must configure the `ShopifyAPI::Context`. The Rails generator will generate a block in the `shopify_app` initializer. To do so manually, you can refer to `after_initialize` block in the [template](https://github.com/Shopify/shopify_app/blob/main/lib/generators/shopify_app/install/templates/shopify_app.rb.tt).

## Upgrading to `v18.1.2`

Version 18.1.2 replaces the deprecated EASDK redirect with an App Bridge 2 redirect when attempting to break out of an iframe. This happens when an app is installed, requires new access scopes, or re-authentication because the login session is expired.

## Upgrading to `v17.2.0`

### Different SameSite cookie attribute behavior

To support Rails `v6.1`, the [`SameSiteCookieMiddleware`](/lib/shopify_app/middleware/same_site_cookie_middleware.rb) was updated to configure cookies to `SameSite=None` if the app is embedded. Before this release, cookies were configured to `SameSite=None` only if this attribute had not previously been set before.

```diff
# same_site_cookie_middleware.rb
- cookie << '; SameSite=None' unless cookie =~ /;\s*samesite=/i
+ cookie << '; SameSite=None' if ShopifyApp.configuration.embedded_app?
```

By default, Rails `v6.1` configures `SameSite=Lax` on all cookies that don't specify this attribute.

## Upgrading to `v13.0.0`

Version 13.0.0 adds the ability to use both user and shop sessions, concurrently. This however involved a large
change to how session stores work. Here are the steps to migrate to 13.x

### Changes to `config/initializers/shopify_app.rb`

- _REMOVE_ `config.per_user_tokens = [true|false]` this is no longer needed
- _CHANGE_ `config.session_repository = 'Shop'` To `config.shop_session_repository = 'Shop'`
- _ADD (optional)_ User Session Storage `config.user_session_repository = 'User'`

### Shop Model Changes (normally `app/models/shop.rb`)

- _CHANGE_ `include ShopifyApp::SessionStorage` to `include ShopifyApp::ShopSessionStorage`

### Changes to the @shop_session instance variable (normally in `app/controllers/*.rb`)

- _CHANGE_ if you are using shop sessions, `@shop_session` will need to be changed to `@current_shopify_session`.

### Changes to Rails `session`

- _CHANGE_ `session[:shopify]` is no longer set. Use `session[:user_id]` if your app uses user based tokens, or `session[:shop_id]` if your app uses shop based tokens.

### Changes to `ShopifyApp::LoginProtection`

`ShopifyApp::LoginProtection`

- CHANGE if you are using `ShopifyApp::LoginProtection#shopify_session` in your code, it will need to be
  changed to `ShopifyApp::LoginProtection#activate_shopify_session`
- CHANGE if you are using `ShopifyApp::LoginProtection#clear_shop_session` in your code, it will need to be
  changed to `ShopifyApp::LoginProtection#clear_shopify_session`

### Notes

You do not need a user model; a shop session is fine for most applications.

---

## Upgrading to `v11.7.0`

### Session storage method signature breaking change

If you override `def self.store(auth_session)` method in your session storage model (e.g. Shop), the method signature has changed to `def self.store(auth_session, *args)` in order to support user-based token storage. Please update your method signature to include the second argument.

---

## Upgrading from `v8.6` to `v9.0.0`

### Configuration change

Add an API version configuration in `config/initializers/shopify_app.rb`
Set this to the version you want to run against by default. See [Shopify API docs](https://help.shopify.com/api/versioning) for versions available.

```ruby
config.api_version = '2019-04'
```

### Session storage change

You will need to add an `api_version` method to your session storage object. The default implementation for this is.

```ruby
def api_version
  ShopifyApp.configuration.api_version
end
```

### Generated file change

`embedded_app.html.erb` the usage of `shop_session.url` needs to be changed to `shop_session.domain`

```erb
<script type="text/javascript">
  ShopifyApp.init({
    apiKey: "<%= ShopifyApp.configuration.api_key %>",

    shopOrigin: "<%= "https://#{ @shop_session.url }" if @shop_session %>",

    debug: false,
    forceRedirect: true
  });
</script>
```

is changed to

```erb
<script type="text/javascript">
  ShopifyApp.init({
    apiKey: "<%= ShopifyApp.configuration.api_key %>",

    shopOrigin: "<%= "https://#{ @shop_session.domain }" if @shop_session %>",

    debug: false,
    forceRedirect: true
  });
</script>
```

### ShopifyAPI changes

You will need to also follow the ShopifyAPI [upgrade guide](https://github.com/Shopify/shopify-api-ruby/blob/master/README.md#-breaking-change-notice-for-version-700-) to ensure your app is ready to work with API versioning.

[dashboard]: https://partners.shopify.com
[app-bridge]: https://shopify.dev/apps/tools/app-bridge
