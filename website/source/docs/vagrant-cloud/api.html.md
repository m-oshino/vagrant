---
layout: "docs"
page_title: "Vagrant Cloud API"
sidebar_current: "vagrant-cloud-api"
---

# Vagrant Cloud API

* [Using the API](#using-the-api)
    * [Authentication](#authentication)
    * [Request and Response Format](#request-and-response-format)
    * [Response Codes](#response-codes)
* [Authentication Tokens](#authentication-tokens)
    * [Create a token](#create-a-token)
    * [Validate a token](#validate-a-token)
* [Boxes](#boxes)
    * [Read a box](#read-a-box)
    * [Create a box](#create-a-box)
    * [Update a box](#update-a-box)
    * [Delete a box](#delete-a-box)
* [Versions](#versions)
    * [Read a version](#read-a-version)
    * [Create a version](#create-a-version)
    * [Update a version](#update-a-version)
    * [Delete a version](#delete-a-version)
    * [Release a version](#release-a-version)
    * [Revoke a version](#revoke-a-version)
* [Providers](#providers)
    * [Read a provider](#read-a-provider)
    * [Create a providers](#create-a-provider)
    * [Update a provider](#update-a-provider)
    * [Delete a provider](#delete-a-provider)
    * [Upload](#upload-a-provider)


## Using the API

Vagrant Cloud provides an API for users to interact with Vagrant Cloud for experimentation, automation, or building new features and tools on top of our existing application.

### Authentication

Some API endpoints require authentication to create new resources, update or delete existing resources, or to read a private resource.

Clients can authenticate using an authentication token.
The token can be passed to Vagrant Cloud one of two ways:

1. (Preferred) Set the `X-Atlas-Token` header to the value of the authentication token.
2. Pass the authentication token as an `access_token` URL parameter.

Examples below will set the header, but feel free to use whichever method is easier for your implementation.

-> The header name `X-Atlas-Token` is an artifact of Vagrant Cloud's previous project name, "Atlas".
-> This header will be updated in the near future, but `X-Atlas-Token` will continue to be supported for now to ensure backwards-compatibility.

### Request and Response Format

Requests to Vagrant Cloud which include data attributes (`POST` or `PUT`/`PATCH`) should set the `Content-Type` header to `"application/json"`, and include a valid JSON body with the request.

JSON responses may include an `errors` key, which will contain an array of error strings, as well as a `success` key.
For example:

```json
{
  "errors": [
    "Resource not found!"
  ],
  "success": false
}
```

### Response Codes

Vagrant Cloud may response with the following response codes, depending on the status of the request and context:

#### Success

##### **200** OK
##### **201** Created

#### Client Errors

##### **401** Unauthorized

You do not have authorization to access the requested resource.

##### **402** Payment Required

You are trying to access a resource which is delinquent on billing.
Please contact the owner of the resource so that they can update their billing information.

##### **403** Forbidden

You are attempting to use the system in a way which is not allowed.
There could be required request parameters that are missing, or one of the parameters is invalid.
Please check the response `errors` key, and double-check the examples below for any discrepancies.

##### **404** Not Found

The resource you are trying to access does not exist. This may also be returned if you attempt to access a private resource that you don't have authorization to view.

##### **429** Too Many Requests

You are currently being rate-limited. Please decrease your frequency of usage, or contact us at [support+vagrantcloud@hashicorp.com](mailto:support+vagrantcloud@hashicorp.com) with a description of your use case so that we can consider creating an exception.

#### Server Errors

##### **500** Internal Server Error

The server failed to respond to the request for an unknown reason.
Please contact [support+vagrantcloud@hashicorp.com](mailto:support+vagrantcloud@hashicorp.com) with a description of the problem so that we can investigate.

##### **503** Service Unavailable

Vagrant Cloud is temporarily in maintenance mode.
Please check the [HashiCorp Status Site](http://status.hashicorp.com) for more information.


## Authentication Tokens

### Create a token

`POST /api/v1/authenticate`

Creates a new token for the given user.

#### Arguments

* `token`
    * `desription` (Optional) - A description of the token.
* `user`
    * `login` - Username or email address of the user authenticating.
    * `password` - The user's password.

#### Example Request

<div class="examples">
  <ul class="examples-header">
    <li class="examples-menu examples-menu-shell"><a onclick="setExampleLanguage('shell');">cURL</a></li>
    <li class="examples-menu examples-menu-ruby"><a onclick="setExampleLanguage('ruby');">Ruby</a></li>
  </ul>
  <div class="examples-body">
    ```shell
    curl \
      --header "Content-Type: application/json" \
      https://app.vagrantup.com/api/v1/authenticate \
      --data '
        {
          "token": {
            "description": "Login from cURL"
          },
          "user": {
            "login": "me",
            "password": "secret"
          }
        }
      '
    ```

    ```ruby
    # gem install http, or add `gem "http"` to your Gemfile
    require "http"

    api = HTTP.persistent("https://app.vagrantup.com").headers(
      "Content-Type" => "application/json"
    )

    response = api.post("/api/v1/authenticate", json: {
      token: { description: "Login from Ruby" },
      user: { login: "me", password: "secret" }
    })

    if response.status.success?
      # Success, the response attributes are available here.
      p response.parse
    else
      # Error, inspect the `errors` key for more information.
      p response.code, response.body
    end
    ```
  </div>
</div>

#### Example Response

```json
{
  "description": "Login from cURL",
  "token": "qwlIE1qBVUafsg.atlasv1.FLwfJSSYkl49i4qZIu8R31GBnI9r8DrW4IQKMppkGq5rD264lRksTqaIN0zY9Bmy0zs",
  "token_hash": "7598236a879ecb42cb0f25399d6f25d1d2cfbbc6333392131bbdfba325eb352795c169daa4a61a8094d44afe817a857e0e5fc7dc72a1401eb434577337d1246c",
  "created_at": "2017-10-18T19:16:24.956Z"
}
```

### Validate a token

`GET /api/v1/authenticate`

Responds [`200 OK`](#200-ok) if the authentication request was successful, otherwise responds [`401 Unauthorized`](#401-unauthorized).

#### Example Request

<div class="examples">
  <ul class="examples-header">
    <li class="examples-menu examples-menu-shell"><a onclick="setExampleLanguage('shell');">cURL</a></li>
    <li class="examples-menu examples-menu-ruby"><a onclick="setExampleLanguage('ruby');">Ruby</a></li>
  </ul>
  <div class="examples-body">
    ```shell
    curl \
      --header "X-Atlas-Token: $VAGRANT_CLOUD_TOKEN" \
      https://app.vagrantup.com/api/v1/authenticate
    ```

    ```ruby
    # gem install http, or add `gem "http"` to your Gemfile
    require "http"

    api = HTTP.persistent("https://app.vagrantup.com").headers(
      "X-Atlas-Token" => ENV['VAGRANT_CLOUD_TOKEN']
    )

    response = api.get("/api/v1/authenticate")

    if response.status.success?
      # Success, the response attributes are available here.
      p response.parse
    else
      # Error, inspect the `errors` key for more information.
      p response.code, response.body
    end
    ```
  </div>
</div>

## Boxes

### Read a box

`GET /api/v1/box/:username/:name`

#### Example Request

<div class="examples">
  <ul class="examples-header">
    <li class="examples-menu examples-menu-shell"><a onclick="setExampleLanguage('shell');">cURL</a></li>
    <li class="examples-menu examples-menu-ruby"><a onclick="setExampleLanguage('ruby');">Ruby</a></li>
  </ul>
  <div class="examples-body">
    ```shell
    curl \
      --header "X-Atlas-Token: $VAGRANT_CLOUD_TOKEN" \
      https://app.vagrantup.com/api/v1/box/me/test
    ```

    ```ruby
    # gem install http, or add `gem "http"` to your Gemfile
    require "http"

    api = HTTP.persistent("https://app.vagrantup.com").headers(
      "X-Atlas-Token" => ENV['VAGRANT_CLOUD_TOKEN']
    )

    response = api.get("/api/v1/box/me/test")

    if response.status.success?
      # Success, the response attributes are available here.
      p response.parse
    else
      # Error, inspect the `errors` key for more information.
      p response.code, response.body
    end
    ```
  </div>
</div>

#### Example Response

```json
{
  "created_at": "2017-10-16T19:29:09.415Z",
  "updated_at": "2017-10-16T19:29:09.415Z",
  "tag": "me/test",
  "name": "test",
  "short_description": "My dev box",
  "description_html": "<p>My development Vagrant box</p>\n",
  "username": "me",
  "description_markdown": "My development Vagrant box",
  "private": true,
  "current_version": null,
  "versions": []
}
```

### Create a box

`POST /api/v1/boxes`

#### Arguments

* `box`
    * `username` - The username of the organization that will own this box.
    * `name` - The name of the box.
    * `short_description` - A short summary of the box.
    * `description` - A longer description of the box. Can be formatted with [Markdown](https://daringfireball.net/projects/markdown/syntax).
    * `is_private` (Optional, default: `true`) - Whether or not this box is private.

#### Example Request

<div class="examples">
  <ul class="examples-header">
    <li class="examples-menu examples-menu-shell"><a onclick="setExampleLanguage('shell');">cURL</a></li>
    <li class="examples-menu examples-menu-ruby"><a onclick="setExampleLanguage('ruby');">Ruby</a></li>
  </ul>
  <div class="examples-body">
    ```shell
    curl \
      --header "Content-Type: application/json" \
      --header "X-Atlas-Token: $VAGRANT_CLOUD_TOKEN" \
      https://app.vagrantup.com/api/v1/boxes \
      --data '
        {
          "box": {
            "username": "me",
            "name": "test",
            "short_description": "My dev box",
            "description": "My development Vagrant box",
            "is_private": true
          }
        }
      '
    ```

    ```ruby
    # gem install http, or add `gem "http"` to your Gemfile
    require "http"

    api = HTTP.persistent("https://app.vagrantup.com").headers(
      "Content-Type" => "application/json",
      "X-Atlas-Token" => ENV['VAGRANT_CLOUD_TOKEN']
    )

    response = api.post("/api/v1/boxes", json: {
      box: {
        username: "me",
        name: "test",
        short_description: "My dev box",
        description: "My development Vagrant box",
        is_private: true
      }
    })

    if response.status.success?
      # Success, the response attributes are available here.
      p response.parse
    else
      # Error, inspect the `errors` key for more information.
      p response.code, response.body
    end
    ```
  </div>
</div>

#### Example Response

Response body is identical to [Reading a box](#read-a-box).

### Update a box

`PUT /api/v1/box/:username/:name`

#### Arguments

* `box`
    * `name` - The name of the box.
    * `short_description` - A short summary of the box.
    * `description` - A longer description of the box. Can be formatted with [Markdown](https://daringfireball.net/projects/markdown/syntax).
    * `is_private` (Optional, default: `true`) - Whether or not this box is private.

#### Example Request

<div class="examples">
  <ul class="examples-header">
    <li class="examples-menu examples-menu-shell"><a onclick="setExampleLanguage('shell');">cURL</a></li>
    <li class="examples-menu examples-menu-ruby"><a onclick="setExampleLanguage('ruby');">Ruby</a></li>
  </ul>
  <div class="examples-body">
    ```shell
    curl \
      --header "Content-Type: application/json" \
      --header "X-Atlas-Token: $VAGRANT_CLOUD_TOKEN" \
      https://app.vagrantup.com/api/v1/box/me/test \
      --request PUT \
      --data '
        {
          "box": {
            "name": "test",
            "short_description": "My dev box",
            "description": "My development Vagrant box",
            "is_private": true
          }
        }
      '
    ```


    ```ruby
    # gem install http, or add `gem "http"` to your Gemfile
    require "http"

    api = HTTP.persistent("https://app.vagrantup.com").headers(
      "Content-Type" => "application/json",
      "X-Atlas-Token" => ENV['VAGRANT_CLOUD_TOKEN']
    )

    response = api.put("/api/v1/box/me/test", json: {
      box: {
        name: "test",
        short_description: "My dev box",
        description: "My development Vagrant box",
        is_private: true
      }
    })

    if response.status.success?
      # Success, the response attributes are available here.
      p response.parse
    else
      # Error, inspect the `errors` key for more information.
      p response.code, response.body
    end
    ```
  </div>
</div>

### Delete a box

`DELETE /api/v1/box/:username/:name`

#### Example Request

<div class="examples">
  <ul class="examples-header">
    <li class="examples-menu examples-menu-shell"><a onclick="setExampleLanguage('shell');">cURL</a></li>
    <li class="examples-menu examples-menu-ruby"><a onclick="setExampleLanguage('ruby');">Ruby</a></li>
  </ul>
  <div class="examples-body">
    ```shell
    curl \
      --header "X-Atlas-Token: $VAGRANT_CLOUD_TOKEN" \
      --request DELETE \
      https://app.vagrantup.com/api/v1/box/me/test
    ```

    ```ruby
    # gem install http, or add `gem "http"` to your Gemfile
    require "http"

    api = HTTP.persistent("https://app.vagrantup.com").headers(
      "X-Atlas-Token" => ENV['VAGRANT_CLOUD_TOKEN']
    )

    response = api.delete("/api/v1/box/me/test")

    if response.status.success?
      # Success, the response attributes are available here.
      p response.parse
    else
      # Error, inspect the `errors` key for more information.
      p response.code, response.body
    end
    ```
  </div>
</div>

#### Example Response

Response body is identical to [Reading a box](#read-a-box).

## Versions

### Read a version
### Create a version
### Update a version
### Delete a version
### Release a version
### Revoke a version

## Providers

### Create a provider
### Read a provider
### Update a provider
### Delete a provider
