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

_Last updated 23 November 2018_

## Introduction

> This "code column" contains code examples. Read the adjacent column for explanation and details.

Shopify provides a powerful ecommerce platform. The platform is robust and full-featured, to the degree that most developers will not wish to build an ecommerce application from scratch. Instead, it is better to use Shopify to build ecommerce sites. If you need functionality or features that are not available in Shopify, or wish to extend Shopify,  the Shopify platform can send *webhooks* to a custom application. Webhooks are HTTP requests that are triggered by visitor actions in Shopify shops, such as adding to a cart or placing an order. Almost all visitor events can trigger a webhook and the webhook will contain a payload of data in a JSON object.

This tutorial is based on work done for the [Retention Rocket](https://www.retentionrocket.com/) platform.

## Scope and Tasks

In this tutorial, you'll learn how to create a Rails API application that receives Shopify webhooks.

A webhook is an HTTP request sent by an application and triggered by a programmatic event. Shopify can be configured to send webhooks for any significant visitor behavior.

If you've searched the web for tutorials about Shopify webhooks, you may have learned that you can create webhooks programmatically using the Shopify API. That means you can build a Rails application, install the [shopify_app gem](https://github.com/Shopify/shopify_app) for Shopify API access, and write code that configures a Shopify store to set up webhooks. Before you start down that path, recognize that you can create Shopify webhooks manually using the  Shopify store admin interface. You may not need to create webhooks programmatically, in which case it is simpler to create Shopify webhooks manually. This tutorial will focus on manual creation of webhooks for simplicity.

You can use the [shopify_app gem](https://github.com/Shopify/shopify_app) to write Rails applications that use the Shopify API. The gem includes a Rails Engine and various generators for controllers, models, and views. The gem also handles authentication and can manage Shopify webhooks. Other tutorials show how to add the [shopify_app gem](https://github.com/Shopify/shopify_app) to a Rails application. With the shopify_app gem you can create storefronts that make API calls for processing by the Shopify API (for example, processing orders and payments). We will not explore the use of the shopify_app gem for creating storefronts. In this tutorial, we'll look at a different use case. We'll build a Rails API application (with no views or user-facing UI) that processes events triggered by visitor actions in a Shopify store. This use case is important for analytics or applications that support or extend basic ecommerce functionality. For example, this approach is used by the [Retention Rocket](https://www.retentionrocket.com/) service to reduce cart abandonment.

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

You can give any URL path you like as a web address, as a substitute for `cart_create` but stick with our example for now. The shopify_app gem expects to see the `webhooks` path as the default route to its Shopify engine controller, so don't change that unless you are prepared to dig deeper into the [shopify_app gem](https://github.com/Shopify/shopify_app) documentation.

Click "Save webhook" and you'll see the webhook listed on the "Settings/Notifications" page in the "Webhooks" section. Notice the link "Send test notification." Launch your local Ngrok client:

`$ ngrok http 3000 -subdomain=my-ngrok-example`

Click "Send test notification" and look for the "502 Bad Gateway" error in the local Ngrok console. You've sent a webhook from Shopify and the Ngrok proxy has forwarded it to the localhost port 3000. No application is available to receive the HTTP request on localhost port 3000 so the Ngrok client reports a "502 Bad Gateway" error. However, if you've gotten this far, you are successful in setting up Shopify to send webhooks to your local development machine. Now we can build a Rails application to respond to the Shopify webhooks.

## Rails API Application
