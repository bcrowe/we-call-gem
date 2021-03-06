# We::Call

[![Build Status][travis-image]][travis-url]
[![Coverage Status][coveralls-image]][coveralls-url]
[![MIT License][license-image]][license-url]

![Call me Maybe](https://cloud.githubusercontent.com/assets/67381/25590846/0c3145ea-2e80-11e7-9166-76448e0134a8.jpeg)

Requires metadata and offers client/server middleware for making HTTP requests, tracking calls, raising deprecations, supporting trace IDs, throttling, etc.

## Goals

- Work just like Faraday out of the box
- Remove some of the guesswork that comes with HTTP service orientated architecures
- Provide sane defaults whenever possible, but ask for more information if required
- Facilitate [HTTP Evolution](https://www.mnot.net/blog/2012/12/04/api-evolution.html)

## Usage


```ruby
gem 'we-call'
```

```ruby
# config/initializers/we-call.rb

We::Call.configure do |config|
  config.app_name = 'service-a'       # default nil (Connection class falls back to APP_NAME or Rails name)
  config.app_env = 'staging'          # default nil (Connection class back to RACK_ENV || RAILS_ENV)
  config.detect_deprecations = false  # default true
end
```

As this is a Faraday wrapper, the only thing that will change from normal Faraday usage is initialization.

```ruby
connection = We::Call::Connection.new(host: 'https://some-service.example.com/', timeout: 5)

# or with a Faraday connection block
connection = We::Call::Connection.new(host: 'https://some-service.example.com/', timeout: 5) do |conn|
  conn.token_auth('abc123token')
  conn.headers['Foo'] = 'bar'
end
```

See more connection block options in the [Faraday documentation](https://github.com/lostisland/faraday).

### Provide an App

An application should provide its own name in the user agent when calling other services. This is important in case this app busts a local cache, causing it to stampeding herd other service(s).

Other services need to know which server is causing the problem, so no connections are allowed through `We::Call` without an app being set.

```ruby
# Provided at config
connection = We::Call.configure do |config|
  config.app_name = 'Service A'
end

# Provided at initialization
connection = We::Call::Connection.new(host: 'https://service-b.example.com/', app: 'Service A', timeout: 5)
```

_Ofc services could lie about this, so do not use App Name for any sort of security. For that you need to use tokens assigned to applications. This is essentially just forcing a user agent._

### Provide an Env

```ruby
# Provided at config
connection = We::Call.configure do |config|
  config.app_env = 'staging'
end

# Provided at initialization
connection = We::Call::Connection.new(host: 'https://service-b.example.com/', env: 'staging', timeout: 5)
```

Not only is knowing the app name important, but knowing the env is necessary too. Sometimes people configure stuff wrong, and Service A (staging) will hit Service B (production) 😨.

If you are using Rack or Rails, you should not need to do this, as it'll use RACK_ENV or RAILS_ENV by default.

### Timeouts

By default Faraday will let HTTP calls go on forever. In reality this is often 30 seconds for e.g: a Heroku app. Asking developers to make a choice about how long they're willing to wait on this call gives them a chance to consider an acceptable timeout.

The lower this number can be the better, as it reduces time web threads spend waiting for calls that are unlikely to respond anyway.

```ruby
# Provided at initialization
connection = We::Call::Connection.new(host: 'https://service-b.example.com/', timeout: 5)
```

Timeouts can only be provided at initialization of a connection, as they should be different for each service. This is down to the sad reality that some internal services are more performant than others, and various third-parties will have different SLAs.

As well as `timeout: num_seconds` which can set the entire open/read (essentially the total response time of the server), another optional argument exists for `open_timeout: numseconds`. This is how long We::Call should spend waiting for a vague sign of life from the server, which by default is 2.

### Deprecations

We::Call helps with a bunch of things, such as the logic to go on client and server side to handle deprecations, both logging calls made to this servies deprecated endpoints, and alerting services when they make calls to deprecated endpoints.

Currently this is done using a simple annotation.

```ruby
# app/controllers/api_controller.rb

extend We::Call::Annotations
```

Then a controller can be annotated as such:

```ruby
# app/controllers/foo_controller.rb

class FooController < ApiController
  +We::Call::Deprecated.new(date: '2018-01-07 00:00:00 EDT')
  def show
    # ...
  end
end
```

It's as simple as that. This annotation will inject a [Sunset header](https://tools.ietf.org/html/draft-wilde-sunset-header-03) and everyone will know its being deprecated.

## Middleware

### Client

**DetectDeprecations**

Automatically enabled, this Faraday middleware will watch for the [Sunset header](https://tools.ietf.org/html/draft-wilde-sunset-header-03) and send warning to `ActiveSupport::Deprecation` if enabled, or to whatever is in `ENV['rake.logger']`.

### Server

**LogUserAgent**

_(Optional)_ Log the User Agent, which might just be browser information (merely kinda handy), or could be an app name, like the one `We::Call::Connection` asks you for.

```ruby
config.middleware.insert_after Rails::Rack::Logger, We::Call::Middleware::Server::LogUserAgent
```

Easy! Check your logs for `user_agent=service-name; app_name=service-name;` The `app_name` will only show up if this was called by `We::Call::Connection` (as this is the only thing setting the `X-App-Name` header.)

## Requirements

- **Ruby:** v2.2 - v2.4
- **Rails:** v4.2 - v5.1
- **Faraday:** v0.8 - v0.13

_For now this gem requires Rails 4.2+ due to some ActiveController functionality we are taking advantage of. Future work will include making this purely rack based._

## TODO

- [ ] Support adding href to Deprecate to make a `Link` with rel=sunset as per Sunset RFC draft 03
- [ ] Remove Rails as a dependency (soft requirement on `ActiveSupport::Deprecated` is fine)
- [ ] Split DetectDeprecations into standalone `faraday-sunset` gem
- [ ] Pass Trace IDs along
- [ ] Work on sane defaults for retries and error raising

## Testing

To run tests and modify locally, you'll want to `bundle install` in this directory.

```
bundle exec rspec
```

## Development

If you want to test this gem within an application, update your Gemfile to have something like this: `gem 'we-call', github: 'wework/we-call', branch: 'BRANCHNAME'` and set your local config: `bundle config --local local.we-call path/to/we-call`

Simply revert the Gemfile change (updating the version as necessary!) and remove the config with `bundle config --delete local.we-call`.

References: [Blog Post](https://rossta.net/blog/how-to-specify-local-ruby-gems-in-your-gemfile.html) and [Bundle Documentation](https://bundler.io/v1.2/git.html#local)

## Contributing

Bug reports and pull requests are welcome on GitHub at [wework/we-call](https://github.com/wework/we-call). This project is intended to be a safe, welcoming space for collaboration, and contributors are expected to adhere to the [Contributor Covenant](http://contributor-covenant.org) code of conduct.


[coveralls-image]:https://coveralls.io/repos/github/wework/we-call/badge.svg?branch=master
[coveralls-url]:https://coveralls.io/github/wework/we-call?branch=master

[travis-url]:https://travis-ci.org/wework/we-js-logger
[travis-image]: https://travis-ci.org/wework/we-js-logger.svg?branch=master

[license-url]: LICENSE
[license-image]: http://img.shields.io/badge/license-MIT-000000.svg?style=flat-square
