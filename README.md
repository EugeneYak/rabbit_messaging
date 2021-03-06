# Rabbit (Rabbit Messaging) &middot;  [![Gem Version](https://badge.fury.io/rb/rabbit_messaging.svg)](https://badge.fury.io/rb/rabbit_messaging) [![Build Status](https://travis-ci.org/umbrellio/rabbit_messaging.svg?branch=master)](https://travis-ci.org/umbrellio/rabbit_messaging) [![Coverage Status](https://coveralls.io/repos/github/umbrellio/rabbit_messaging/badge.svg?branch=master)](https://coveralls.io/github/umbrellio/rabbit_messaging?branch=master)

Provides client and server support for RabbitMQ

## Installation

```ruby
gem "rabbit_messaging"
```

```shell
$ bundle install
# --- or ---
$ gem install "rabbit_messaging"
```

```ruby
require "rabbit_messaging"
```

## Usage

- [Configuration](#configuration)
- [Client](#client)
- [Server](#server)

---

### Configuration

- RabbitMQ connection configuration fetched from the `bunny_options` section
  of `/config/sneakers.yml`

- `Rabbit.config` provides setters for following options:

  * `group_id` (`Symbol`), *required*

    Shared identifier which used to select api. As usual, it should be same as default project_id
    (I.e. we have project 'support', which runs only one application in production.
    So on, it's group_id should be :support)

  * `project_id` (`Symbol`), *required*

    Personal identifier which used to select exact service.
    As usual, it should be same as default project_id with optional stage_id.
    (I.e. we have project 'support', in production it's project_id is :support,
    but in staging it uses :support1 and :support2 ids for corresponding stages)

  * `hooks` (`Hash`)

    :before_fork and :after_fork hooks, used same way as in unicorn / puma / que / etc

  * `environment` (one of `:test`, `:development`, `:production`), *default:* `:production`

    Internal environment of gem.

      * `:test` environment stubs publishing and does not suppress errors
      * `:development` environment auto-creates queues and uses default exchange
      * `:production` environment enables handlers caching and gets maximum strictness

  * `receiving_job_class_callable` (`Proc`)

    Custom ActiveJob subclass to work with received messages.

  * `exception_notifier` (`Proc`)

    By default, exceptions are reported using `ExceptionNotifier` (see exception_notification gem).
    You can provide your own notifier like this:

    ```ruby
      config.exception_notifier = proc { |e| MyCoolNotifier.notify!(e) }
    ```

---

### Client

```ruby
Rabbit.publish(
  routing_key: :support,
  event: :ping,
  data: { foo: :bar }, # default is {}
  exchange_name: 'fanout', # default is fine too
  confirm_select: true, # setting this to false grants you great speed up and absolutelly no guarantees
  headers: { "foo" => "bar" }, # custom arguments for routing, default is {}
)
```

- This code sends messages via basic_publish with following parameters:

  * `routing_key`: `"support"`
  * `exchange`: `"group_id.project_id.fanout"` (default is `"group_id.poject_id"`)
  * `mandatory`: `true` (same as confirm_select)

    It is set to raise error if routing failed

  * `persistent`: `true`
  * `type`: `"ping"`
  * `content_type`: `"application/json"` (always)
  * `app_id`: `"group_id.project_id"`

- Messages are logged to `/log/rabbit.log`

---

### Server

- Server is supposed to run inside a daemon via the `daemons-rails` gem. Server is run with
`Rabbit::Daemon.run`. `before_fork` and `after_fork` procs in `Rabbit.config` are used
to teardown and setup external connections between daemon restarts, for example ORM connections

- After the server runs, received messages are handled by `Rabbit::EventHandler` subclasses.
  Subclasses are selected by following code:
  ```ruby
    "rabbit/handler/#{group_id}/#{event}".camelize.constantize
  ```

  They use powerful `Tainbox` api to handle message data. Project_id also passed to them.

  If you wish so, you can override `initialize(message)`, where message is an object
  with simple api (@see lib/rabbit/receiving/message.rb)

  Handlers can specify a queue their messages will be put in via a `queue_as` class macro (accepts
  a string / symbol / block with `|message, arguments|` params)

- Received messages are logged to `/log/sneakers.log`, malformed messages are logged to
`/log/malformed_messages.log` and deleted from queue

---

## Contributing

Bug reports and pull requests are welcome on GitHub at https://github.com/umbrellio/rabbit_messaging.

## License

Released under MIT License

## Authors

Team Umbrellio

---

<a href="https://github.com/umbrellio/">
<img style="float: left;" src="https://umbrellio.github.io/Umbrellio/supported_by_umbrellio.svg"
alt="Supported by Umbrellio" width="439" height="72">
</a>
