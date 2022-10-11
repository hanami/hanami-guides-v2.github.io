---
sidebar_position: 1
---

# Containers and dependencies

In Hanami, the application code you add to your `/app` directory is automatically organised into a container. This container forms the basis of a depenency injection system, in which the dependencies of the components you create are provided to them automatically.

Let's take a look at what this means in practice.

Imagine we're building a Bookshelf notifications service for sending notifications to users of the Bookshelf platform. After running `hanami new notifications_service`, our first task is to send welcome emails. To achieve this, we want to provide a `POST /welcome-emails` action that will send a welcome email, probably via a send welcome operation, which in turn will want to render our email in both html and plain text.

As a first pass, we might add four Ruby classes to our `/app` folder - our action, operation, and two renderers.

On the file system, this might look like:

```shell
app
├── actions
│   └── welcome_emails
│       └── create.rb
└── emails
    └── welcome
        ├── operations
        │   └── send.rb
        └── renderers
            ├── html.rb
            └── text.rb
```

When our application boots, Hanami will automatically create instances of these components and register them in its __app container__, under a key based on their Ruby namespace.

For example, an instance of our `NotificationsService::Emails::Welcome::Operations::Send` class will be registered under the key `"emails.welcome.operations.send"`.

```ruby title="app/emails/welcome/operations/send.rb"
# frozen_string_literal: true

module NotificationService
  module Emails
    module Welcome
      module Operations
        class Send
          def call(name:, email_address:)
            puts "Sending greetings to #{name} via #{email_address}!"
          end
        end
      end
    end
  end
end
```

We can see this in the Hanami console if we boot our application and ask what keys are registered with the app container:

```ruby
bundle exec hanami console

notifications_service[development]> Hanami.app.boot
=> NotificationsService::App

notifications_service[development]> Hanami.app.keys
=> ["notifications",
 "settings",
 "routes",
 "inflector",
 "logger",
 "rack.monitor",
 "actions.welcome_emails.create",
 "emails.welcome.operations.send",
 "emails.welcome.renderers.html",
 "emails.welcome.renderers.text"]
 ```

To fetch our welcome email send operation from the container, we ask for it by its `"emails.welcome.operations.send"` key:

```ruby
notifications_service[development]> Hanami.app["emails.welcome.operations.send"]
=> #<NotificationsService::Emails::Welcome::Operations::Send:0x000000010df0a1a0>

notifications_service[development]> Hanami.app["emails.welcome.operations.send"].call(name: "New user", email_address: "email@example.com")
Sending greetings to New user email@example.com!
```

Most of the time however, you won't use the container directly via `Hanami.app`, but will instead make use of the container through the dependency injection system it supports. Let's see how that works!

## Dependency injection

Dependency injection is a software pattern where, rather than a component knowing how to instantiate its dependencies, the dependencies are instead provided to it. This means its dependencies can be abstract rather than hard coded, making the component more flexible, reusable and easier to test.

To illustrate, here's an example of a welcome email send operation which _doesn't_ use dependency injection:

```ruby title="app/emails/welcome/operations/send.rb"
# frozen_string_literal: true

require "acme_email/client"

module NotificationsService
  module Emails
    module Welcome
      module Operations
        class Send
          def call(name:, email_address:)
            return unless Hanami::Settings.new.email_sending_enabled

            AcmeEmail::Client.new.deliver(
              to: email_address,
              subject: "Welcome!",
              text_body: Renderers::Text.new.call(name: name),
              html_body: Renderers::Html.new.call(name: name)
            )
          end
        end
      end
    end
  end
end
```

This component has four dependencies, each of which is a "hard coded" reference to a concrete Ruby class:

- `Hanami::Settings`, used to check whether email sending is enabled in the current environment.
- `AcmeEmail::Client`, used to queue the email for delivery via the third party Acme Email service.
- `Renderers::Text`, used to render the text version of the welcome email.
- `Renderers::Html`, used to render the html version of the welcome email.

To make our send welcome email operation more resuable and easier to test, we could instead _inject_ its dependencies when we initialize it:

```ruby title="app/emails/welcome/operations/send.rb"
# frozen_string_literal: true

module NotificationsService
  module Emails
    module Welcome
      module Operations
        class Send
          attr_reader :email_client
          attr_reader :settings
          attr_reader :text
          attr_reader :html

          def initialize(email_client:, settings:, text:, html:)
            @email_client = email_client
            @settings = settings
            @text = text
            @html = html
          end

          def call(name:, email_address:)
            return unless settings.email_sending_enabled

            email_client.deliver(
              to: email_address,
              subject: "Welcome!",
              text_body: text.call(name: name)
              html_body: html.call(name: name)
            )
          end
        end
      end
    end
  end
end
```

As a result of injection, our component no longer has rigid dependencies - it's able to use any email client, settings object or renderers we provide.

Hanami makes this style of dependency injection simple through an `include Deps[]` mechanism. Built into the app container (and all slice containers), `include Deps[]` allows a component to use any other component in its container as a dependency, while removing the need for any attr_reader or initializer boilerplate:

```ruby title="app/emails/welcome/operations/send.rb"
# frozen_string_literal: true

module NotificationsService
  module Emails
    module Welcome
      module Operations
        class Send
          include Deps[
            "settings",
            "email_client",
            "emails.welcome.renderers.text",
            "emails.welcome.renderers.html"
          ]

          def call(name:, email_address:)
            return unless settings.email_sending_enabled

            email_client.deliver(
              to: email_address,
              subject: "Welcome!",
              text_body: text.call(name: name),
              html_body: html.call(name: name)
            )
          end
        end
      end
    end
  end
end
```

## Injecting dependencies via `include Deps[]`

In the above example the `include Deps[]` mechanism takes each given key and makes the relevant component from the app container available via an instance method of the same name. i.e. `include Deps["settings"]` makes the `settings` registration from the app container available anywhere in the class via the `#settings` method. By default, dependencies are made available under a method named after the last segment of their key. So `include Deps["emails.welcome.renderers.html"]` makes the html renderer available via the method `#html`.

We can see `include Deps[]` in action in the console if we instantiate an instance of our send welcome email operation:

```ruby
notifications_service[development]> NotificationsService::Emails::Welcome::Operations::Send.new
=> #<NotificationsService::Emails::Welcome::Operations::Send:0x000000010c843660
 @email_client=#<AcmeEmail::Client:0x000000010c808858>,
 @html=#<NotificationsService::Emails::Welcome::Renderers::Html:0x000000010c843818>,
 @settings=
  #<NotificationsService::Settings:0x000000010c4bf340
   @config=#<Dry::Configurable::Config values={:email_sending_enabled=>true}>>,
 @text=#<NotificationsService::Emails::Welcome::Renderers::Text:0x000000010c858628>>
```

We can provide different dependencies during initialization:

```ruby
notifications_service[development]> NotificationsService::Emails::Welcome::Operations::Send.new(email_client: "another client")
=> #<NotificationsService::Emails::Welcome::Operations::Send:0x000000010c1ded88
 @email_client="another client",
 @html=#<NotificationsService::Emails::Welcome::Renderers::Html:0x000000010c1df1c0>,
 @settings=
  #<NotificationsService::Settings:0x000000010c4bf340
   @config=#<Dry::Configurable::Config values={:email_sending_enabled=>true, :acme_api_key=>"sdf"}>>,
 @text=#<NotificationsService::Emails::Welcome::Renderers::Text:0x000000010c28ca50>>
```

This behaviour is particularly useful when testing, as you can substitute one or more components to test behaviour.

In this unit test, we substitute each of the operation's dependencies in order to unit test its behaviour:

```ruby title="spec/unit/emails/welcome/operations/send_spec.rb"
RSpec.describe NotificationsService::Emails::Welcome::Operations::Send, "#call" do
  subject(:send) {
    described_class.new(
      email_client: email_client,
      settings: settings,
      text: text,
      html: html
    )
  }

  let(:email_client) { double(:email_client) }
  let(:text) { double(:text, call: "Welcome to Bookshelf Bookshelf user") }
  let(:html) { double(:html, call: "<p>Welcome to Bookshelf Bookshelf user</p>") }

  context "when email sending is enabled" do
    let(:settings) { double(:settings, email_sending_enabled: true) }

    it "delivers an email using the email client" do
      expect(email_client).to receive(:deliver).with(
        to: "email@example.com",
        subject: "Welcome!",
        text_body: "Welcome to Bookshelf Bookshelf user",
        html_body: "<p>Welcome to Bookshelf Bookshelf user</p>"
      )

      send.call(name: "Bookshelf user", email_address: "email@example.com")
    end
  end

  context "when email sending is not enabled" do
    let(:settings) { double(:settings, email_sending_enabled: false) }

    it "does not deliver an email" do
      expect(email_client).not_to receive(:deliver)

      send.call(name: "Bookshelf user", email_address: "email@example.com")
    end
  end
end
```

Exactly which dependency to stub using RSpec mocks is up to you - if a depenency is left out of the constructor within the spec, then the real dependency is resolved from the container. Every test can decide exactly which dependencies to replace.

## Renaming dependencies

Sometimes you want to use a dependency under another name (either because two dependencies end with the same suffix, or just because it makes things clearer in a different context).

This can be done with `include Deps[]` like so:

```ruby
module NotificationsService
  class NewBookNotification
    include Deps[
      "settings",
      send_email_notification: "emails.book_added.operations.send",
      send_slack_notification: "slack_notifications.book_added.operations.send"
    ]

    def call(...)
      send_email_notification.call(...) if settings.email_sending_enabled
      send_slack_notification.call(...) if settings.slack_sending_enabled
    end
  end
end
```

## Opting out of the container

Sometimes it doesn’t make sense for something to be put in the container. For example, Hanami provides a base action class at `app/action.rb` from which all actions inherit. This type of class will never be used as a dependency by anything, and so registering it in the container doesn’t make sense.

For once-off exclusions like this Hanami supports a magic comment: `# auto_register: false`

```ruby title="app/action.rb"
# auto_register: false
# frozen_string_literal: true

require "hanami/action"

module NotificationsService
  class Action < Hanami::Action
  end
end
```

Another alternative for classes you do not want to be registered in your container is to place them in `/lib`.

If you have a whole class of objects that shouldn't be placed in your container, you can configure your Hanami application (or slice) to exclude an entire directory from auto registration by adjusting its `no_auto_register_paths` configuration.

Here for example, the `app/structs` directory is excluded:

```ruby title="config/app.rb"
# frozen_string_literal: true

require "hanami"

module NotificationsService
  class App < Hanami::App
    config.no_auto_register_paths << "structs"
  end
end
```

## Container behaviour: prepare vs boot

Hanami supports a **prepared** state and a **booted** state.

### Hanami.prepare

When you call `Hanami.prepare` (or use `require "hanami/prepare"`) Hanami will make its app and slices available, but components within containers will be **lazily loaded**.

This is useful for minimizing load time. It's the default mode in the Hanami console and when running tests.

It can also be very useful when running Hanami in serverless environments where boot time matters, such as on AWS Lambda, as Hanami will instantiate only the components needed to satisfy a particular web request or operation.

### Hanami.boot

When you call `Hanami.boot` (or use `require "hanami/boot"`) Hanami will go one step further and **eagerly load** all components in all containers up front.

This is useful in contexts where you want to incur initialization costs up front, such as when preparing your application to serve web requests. It's the default when running via Hanami's puma setup (see `config.ru`).