---
title: Shopify Webhooks

language_tabs: # must be one of https://git.io/vQNgJ

toc_footers:
  - This document is open source
  - Questions? <a href='https://stackoverflow.com/questions/tagged/ruby-on-rails'>Ask on Stack Overflow</a>
  - Have fixes? <a href='https://github.com/RetentionRocket/tutorial-shopify-webhooks'>Edit the GitHub repo</a>


includes:

search: true
---

# Shopify Webhooks

**by Daniel Kehoe**

_Last updated 24 November 2018_

## Introduction

> This "code column" contains code examples. Read the adjacent column for explanation and details.

Shopify provides a powerful ecommerce platform. The platform is robust and full-featured, to the degree that most developers will not wish to build an ecommerce application from scratch. Instead, many developers use Shopify to build ecommerce sites. If you need functionality or features that are not available in Shopify, or wish to extend Shopify,  the Shopify platform can send *webhooks* to a custom application. Webhooks are HTTP requests that are triggered by visitor actions in Shopify shops, such as adding to a cart or placing an order. Almost all visitor events can trigger a webhook and the webhook will contain a payload of data in a JSON object that you can process in another application.

This tutorial is based on work done for the [Retention Rocket](https://www.retentionrocket.com/) platform.

## Scope and Tasks

In this tutorial, you'll learn how to create a Rails API application that receives Shopify webhooks.

A webhook is an HTTP request sent by an application and triggered by a programmatic event. Shopify can be configured to send webhooks for any significant visitor behavior.

You can use the [shopify_app gem](https://github.com/Shopify/shopify_app) to write Rails applications that handle Shopify webhooks. The gem handles the necessary authentication and can manage Shopify webhooks. The gem is primarily intended to enable custom storefronts to make API calls for processing by the Shopify API (for example, processing orders and payments). The gem includes a Rails Engine and various generators for controllers, models, and views. Other tutorials show how to add the shopify_app gem to a Rails application for that purpose. We will not explore the use of the shopify_app gem for creating storefronts. In this tutorial, we'll look at a different use case. We'll build a Rails API application (with no views or user-facing UI) that processes events triggered by visitor actions in a Shopify store. This use case is important for analytics or applications that support or extend basic ecommerce functionality. For example, this approach is used by the [Retention Rocket](https://www.retentionrocket.com/) service to reduce cart abandonment.

If you've searched the web for tutorials about Shopify webhooks, you may have learned that you can create webhooks programmatically using the Shopify API and the shopify_app gem. That means you can build a Rails application, install the shopify_app gem for Shopify API access, and write code that configures a Shopify store to set up webhooks via API calls. Before you start down that path, recognize that you can create Shopify webhooks manually using the Shopify store admin interface. You may not need to create webhooks programmatically, in which case it is simpler to create Shopify webhooks manually. This tutorial will focus on manual creation of webhooks for simplicity.

First you'll set up a "tunnel" using [Ngrok](https://ngrok.com/) so Shopify can send HTTP requests to your local development machine. Then you'll create a Shopify partner account, a development store, and create a webhook for testing. We'll show you how to build a Rails API application that will respond to Shopify HTTP requests and you can send test webhooks from your development store and observe the HTTP requests in Ngrok and your local server console.

## Tunnel to Localhost

The Shopify application can send HTTP requests to any web address you specify as a webhook "endpoint" or destination. Notably, that doesn't include servers running on "localhost" because most local development machines are not exposed to the public Internet with a fixed IP address (Internet service providers typically assign IP addresses dynamically and the addresses are not publicly advertised). That means you must run a program on your local machine that sets up a "tunnel" from a localhost port to a public proxy service that can receive and forward HTTP requests. Several services are available (see [A Survey of the Localhost Proxying Landscape](https://john-sheehan.com/blog/a-survey-of-the-localhost-proxying-landscape)). We'll use [Ngrok](https://ngrok.com/) as it is the most popular and allows monitoring ("introspection") of the HTTP requests passing through the tunnel.

You can use [Ngrok for free](https://ngrok.com/pricing) and it will assign a public URL randomly each time you run the ngrok application. It's more convenient to sign up for an account at a cost of $5/month and set up a reserved subdomain. If you're working for someone who already has a paid Ngrok account, ask them to [invite you as a team member](https://dashboard.ngrok.com/team). Sign up for an Ngrok account using GitHub authentication (or use an email address and password). Set up a [reserved domain](https://dashboard.ngrok.com/reserved) in Ngrok, for example,  [my-ngrok-example.ngrok.io](https://my-ngrok-example.ngrok.io) in the United States region. Later, you'll set this web address as an endpoint for a webhook in your Shopify development store.

[Download and set up the Ngrok client](https://ngrok.com/download). You'll need to visit the Ngrok dashboard and get an authentication token and run a one-time command that adds the authentication token to a configuration file. Then you can launch the Ngrok client to open a tunnel from a localhost port to the Ngrok public proxy. You'll enter a command like this to launch the Ngrok client:

`$ ngrok http 3000 -subdomain=my-ngrok-example`

This will allow you to send HTTP requests to [https://my-ngrok-example.ngrok.io/](https://my-ngrok-example.ngrok.io/).

The Ngrok client is launched from your terminal and displays any incoming HTTP requests in the console. A local web interface is available at [http://127.0.0.1:4040](http://127.0.0.1:4040) for inspecting the incoming HTTP requests.

Try the Ngrok destination URL in a browser:

* [https://my-ngrok-example.ngrok.io/](https://my-ngrok-example.ngrok.io/)

You'll see a "502 Bad Gateway" error in the console running the Ngrok client. The web interface at [http://127.0.0.1:4040](http://127.0.0.1:4040) will show details of the failed request. Your browser will display a page with the message "Failed to complete tunnel connection" because there is no web application server responding to HTTP requests on your localhost port 3000. We'll need to build a Rails application to respond to HTTP requests forwarded by Ngrok.

Now that Ngrok is set up in the middle to receive HTTP requests from Shopify and pass them to a localhost application, you'll need to set up a Shopify development store to send HTTP requests to Ngrok. You'll also need to create a local Rails application to receive the HTTP requests from the Ngrok proxy. If you launched the Ngrok client already, exit with control-c and launch it again when everything is ready.

## Shopify Partner Account

You'll need a Shopify partner account to create a development store and set up webhooks. Shopify primarily targets designers and developers of ecommerce stores for the partner program. In this context, you're a third-party developer but you'll still need a Shopify partner account. Visit the Shopify site to [create a Shopify partner account](https://www.shopify.ca/partners).

## Create a Development Store

Log in to your Shopify partner account to [create a development store](https://help.shopify.com/en/partners/dashboard/development-stores). You'll click "Development stores" and then click "Create store."

Note that you must use your account email as the login for the development store. If you are a collaborator or staff on someone else's account, the login cannot be changed and the account owner will get notifications generated by the development store. It's not clear why Shopify requires the account owner's login and why it cannot be changed.

You can "Add product" to the store, or make any other changes you like, but it is not necessary if you just want to test webhooks.

## Test Webhooks

We've set up a development store so we can generate and test webhooks. Look for the "Settings" link in the lower left corner of the Shopify store admin page ([https://example-store.myshopify.com/admin](https://example-store.myshopify.com/admin)). The "Settings" link is easy to overlook in the lower left corner. Click "Settings" and look for the "Notifications" link.

Scroll to the bottom of the "Settings/Notifications" page. The "Webhooks" section is at the bottom of the page. Click the button to "Create webhook." For testing, choose "Cart creation" as the event with a "JSON" format.

You must provide a URL that will respond to the HTTP request that Shopify will send when you test the webhook. You've already set up Ngrok with a reserved subdomain. Enter the web address:

`https://my-ngrok-example.ngrok.io/webhooks/cart_create`

You can give any URL path you like as a web address as a substitute for `cart_create` but stick with our example for now. The shopify_app gem expects to see the `webhooks` path as the default route to its Shopify engine controller, so don't change that unless you are prepared to dig deeper into the [shopify_app gem](https://github.com/Shopify/shopify_app) documentation.

Click "Save webhook" and you'll see the webhook listed on the "Settings/Notifications" page in the "Webhooks" section.

Make a note of the API key that is displayed below the webhook (after the text "All your webhooks will be signed with..."). When we create a Rails application, you'll need to create a Shopify initializer file (or Unix environment variable) to provide this authentication key to the Shopify_app engine.

 Notice the link "Send test notification." Launch your local Ngrok client:

`$ ngrok http 3000 -subdomain=my-ngrok-example`

Click "Send test notification" and look for the "502 Bad Gateway" error in the local Ngrok console. You've sent a webhook from Shopify and the Ngrok proxy has forwarded it to the localhost port 3000. No application is available to receive the HTTP request on localhost port 3000 so the Ngrok client reports a "502 Bad Gateway" error. However, if you've gotten this far, you are successful in setting up Shopify to send webhooks to your local development machine. Now we can build a Rails application to respond to the Shopify webhooks.

## Rails API Application

If you're new to Rails, you can learn [how to install Rails](http://railsapps.github.io/installrubyonrails-mac.html) and set up your development environment. Let's check that Ruby and Rails are installed. You'll need Ruby 2.5 (or newer) and Rails 5.2 (or newer) on your computer to follow this tutorial.

`$ ruby -v`

`$ rails -v`

### New Rails API Application

Generate a simple Rails default starter application:

`$ rails new shopify-webhooks --api`

You can give the application any name you like. We'll call it `shopify-webhooks`. The `--api` argument is available in Rails 5.0 (and newer) and generates an API-centric application without view files and other code required for a user-facing interface.

A Rails API application is faster and contains less code than a conventional Rails application. However, the authentication strategy implemented by the shopify_app gem requires cookies for session persistence. Session persistence is not implemented by default in a Rails API application so we'll add it by modifying the application.

### Shopify Gem

Open the Gemfile and add the [shopify_app gem](https://github.com/Shopify/shopify_app) at the end of the file:

`gem 'shopify_app'`

Install the gem with Bundler.

`$ bundle install`

The [README](https://github.com/Shopify/shopify_app) documentation for the shopify_app gem explains how to use shopify_app generators to create configuration and view files. We are building an API-only application, so we won't use the shopify_app `shopify_app:install` generator. We'll use the generator to create a `Shop` model and then create the necessary files manually.

### Shop Model

The shopify_app engine requires a `Shop` model as part of the authentication strategy.

Use the shopify_app generator to create a `Shop` model:

`$ rails generate shopify_app:shop_model`

The generator will create a file *app/models/shop.rb* and modify the file *config/initializers/shopify_app.rb*.

```ruby
class Shop < ApplicationRecord
  include ShopifyApp::SessionStorage
end
```

This model includes the `ShopifyApp::SessionStorage` concern from the shopify_app gem which adds two methods to make it compatible as a `SessionRepository`.

The shopify_app generator will create a database migration:

```ruby
class AddShopifyToShops < ActiveRecord::Migration[5.1]
  def change
    add_column :shops, :shopify_domain, :string, null: false
    add_column :shops, :shopify_token, :string, null: false
    add_index :shops, :shopify_domain, unique: true
  end
end
```

Run the migration to create the Shops table:

`$ rails db:migrate`

### Modify *config/application.rb*

The shopify_app engine requires cookie session storage and access to Rails Sprockets. A Rails application generated with the `--api` argument doesn't contain session storage or Rails Sprockets so we'll add as necessary. Modify the file *config/application.rb*:

Uncomment:

`require "sprockets/railtie"`

Add:

`config.middleware.use ActionDispatch::Cookies`

`config.middleware.use ActionDispatch::Session::CookieStore`

### Modify *config/routes.rb*

Add the shopify_app engine:

`mount ShopifyApp::Engine, at: '/'`

### CheckoutsUpdateJob

Create a file *app/jobs/checkouts_update_job.rb*.

```ruby
class CheckoutsUpdateJob < ActiveJob::Base
  def perform(shop_domain:, webhook:)
    Rails.logger.info "CheckoutsUpdateJob"
  end
end
```

### OmniAuth Initializer

<aside class="warning">
TODO: Is this needed?
</aside>

Add a file `config/initializers/omniauth.rb`:

```ruby
Rails.application.config.middleware.use OmniAuth::Builder do
  provider :shopify, ENV['SHOPIFY_API_KEY'], ENV['SHOPIFY_API_SECRET'],
    scope: 'read_customers',
    setup: lambda { |env| params = Rack::Utils.parse_query(env['QUERY_STRING'])
                        env['omniauth.strategy'].options[:client_options][:site] = "http://#{params['shop']}" }
end
```

### Shopify Initializer

<aside class="warning">
TODO: Is 'config.session_repository = Shop' needed?
</aside>

Add a file `config/initializers/shopify_app.rb`:

```ruby
ShopifyApp.configure do |config|
  config.application_name = "RetentionRocket"
  config.api_key = ENV["SHOPIFY_API_KEY"]
  config.secret = ENV["SHOPIFY_API_SECRET"]
  config.scope = "read_orders, read_products"
  config.embedded_app = true
  config.after_authenticate_job = false
  config.session_repository = Shop
end
```

### Set Credentials

The Shopify engine only responds to webhook HTTP requests if it can authenticate the request.

For the best security, it is advisable to keep API credentials out of your code so they won't become publicly visible in a GitHub repository. To that end, we've set up the Shopify initializer file to obtain Shopify credentials from Unix environment variables. You'll need to set two Unix environment variables:

* SHOPIFY_API_KEY
* SHOPIFY_API_SECRET

You can find the SHOPIFY_API_KEY on the Shopify development store "Settings/Notification" page at [https://example-store.myshopify.com/admin/settings/notifications](https://example-store.myshopify.com/admin/settings/notifications). It is not labeled as an API key but it appears after the text "All your webhooks will be signed with...".

Add your Shopify credentials to your *.bash_profile* or *.bashrc* file:

`export SHOPIFY_API_KEY="some_long_string"`

`export SHOPIFY_API_SECRET="another_long_string"`

<aside class="warning">
TODO: Is the SHOPIFY_API_SECRET needed? Where do we find it?
</aside>

Close and reopen your terminal to make sure the environment is updated with the recent changes.

### Shopify Session Initializer

<aside class="warning">
TODO: Is this needed?
</aside>

Add a file `config/initializers/shopify_session_repository.rb`:

```ruby
if Rails.configuration.cache_classes
  ShopifyApp::SessionRepository.storage = Shop
else
  reloader = defined?(ActiveSupport::Reloader) ? ActiveSupport::Reloader : ActionDispatch::Reloader

  reloader.to_prepare do
    ShopifyApp::SessionRepository.storage = Shop
  end
end
```
