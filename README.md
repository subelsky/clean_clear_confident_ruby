# Writing Clean, Concise, and Confident Ruby Code

Ideas for writing code that is easy to change,
easy to understand, easy to debug,
and fun to work on.

* mike@subelsky.com
* http://twitter.com/subelsky
* http://tinyletter.com/subelsky

[Presentation software](https://github.com/sotte/presenting.vim)

## Adopt confident code conventions

* http://www.confidentruby.com
* http://exceptionalruby.com

## Master object-oriented design

* http://c2.com/cgi/wiki?PatternsForBeginners
* http://www.amazon.com/Design-Patterns-Ruby-Russ-Olsen/dp/0321490452

## Remember you & your descendants will have to live with your code forever

## Keep code modules shorter than one screenful

```ruby
  require "workers/mission_datapoint_deleter"

  ## Ensures the removal of datapoints collected for missions that only partially succeeded.
  ## Important because of subcollectors: one collection mission could involve many different
  ## Sidekiq jobs occuring on multiple servers at multiple times.
  ## @see CollectionBatchChecker
  class DatapointCleaner
    ## @param mission [CollectionMission] the mission that needs to be checked
    def clean(mission)
      if mission.failed? || mission.expired?
        MissionDatapointDeleter.perform_async(mission.id,mission.attributes.only("state","message"))
      end
    end
  end
```

## Avoid conditionals, prefer polymorphism

### events/lib/services/event_handler.rb

```ruby
  require_relative "../events"

  ## Looks for an event handler for an event with the given name. If none is found, ignores
  ## the event. Otherwise instantiates the handler calls .handle with the given payload.
  class EventHandler
    attr_writer :logger

    ## @param event_name [String] name of the event whose handler should be invoked
    ## @param payload anything else to pass to the handler
    def perform(event_name,*payload)
      logger.info "Received #{event_name} with payload #{payload.inspect}"
      return unless Module.const_defined?(event_name)
      handler_class = Module.const_get(event_name)
      handler_class.new.handle(*payload)
    end

    private

    def logger
      @logger ||= AppContainer.logger
    end
  end
```

## Write lots of tests

* http://www.amazon.com/Growing-Object-Oriented-Software-Guided-Tests/dp/0321503627

### spec/lib/services/event_handler_spec.rb

```ruby
require "spec_helper"
require "services/event_handler"

describe EventHandler do
  let(:result) { double(:result,success?: true) }
  let(:handler) { double(:handler,handle: nil) }
  let(:handler_class) { double(:hc,new: handler) }

  it "passes row ID to the event handler" do
    Module.const_set("GenericEvent",handler_class)
    expect(handler).to receive(:handle).with(10)
    subject.perform("GenericEvent",10)
  end

  it "ignores unrecognized events" do
    expect {
      subject.perform("MadeUpNothingness",10)
    }.not_to raise_error
  end
end
```

### spec/integration/responses/admin_inserting_spec.rb

```ruby
require "spec_helper"
require "events"

describe AdminInsertEvent do
  let(:admin) { create(:admin,email: "newperson@staq.com") }

  include Mail::Matchers

  before do
    Mail::TestMailer.deliveries.clear
  end

  it "emails a welcome message" do
    event_handler.perform("AdminInsertEvent",admin.id)
    expect(subject).to have_sent_email.to("newperson@staq.com").from("info@staq.com").matching_subject(/welcome/i)
    expect(Mail::TestMailer).to have(1).deliveries
  end
end
```

## Make logging easy

## Use dependency injection

* Doesn't have to be fancy
* http://github.com/subelsky/dim

```ruby
require "logger"

AppContainer = Dim::Container.new

AppContainer.register(:env) { ENV.fetch("STAQ_ENV") }

AppContainer.register(:logger) do |c|
  if c.test?
    Logger.new("#{c.root}/log/#{c.env}.log")
  else
    Logger.new(STDOUT)
  end
end
```

## && and || can be code smells

```ruby
if current_user && current_user.authorized?
  ## do something if both are true
else
  redirect_to root_url, error: "You are not authorized"
end
```

To thoroughly test that behavior, I need at least 2 tests. This is possibly better:

```ruby
  UserAuthorizer.call(current_user,name_of_current_action) do
    logger.warn "Something shady happened ..."
    redirect_to root_url, error: "You are not authorized"
    return
  end

  # proceed with normal behavior
```

## Avoid nil-checking, use null objects or code blocks

* http://devblog.avdi.org/2011/05/30/null-objects-and-falsiness/
* http://www.omniref.com/ruby/gems/activesupport/4.0.2/classes/Object##presence
* https://github.com/avdi/naught

```ruby
class User < ActiveRecord::Base
  ## has a database column called "name" that the Rails
  ## app expects to be able to call
end

class GuestUser
  ## implements same behavior as a logged-in user,
  ## means we don't have to check if logged_in_user.nil?
  def name
    "Guest"
  end
end
```

## When dealing with strings, prefer .blank? vs. .nil?

```ruby
  ## require "active_support/core_ext/object/blank"

  if value.blank?
    ## handle case where value is '', ' ', or nil
  end
```

## Keep models, controllers & workers thin/simple

* http://en.wikipedia.org/wiki/Single_responsibility_principle
* Service objects
* Proprietary gems
* Serializers

### web/app/controllers/exports_controller.rb

```ruby
class ExportsController < ApplicationController
  before_filter :find_applet

  def create
    email = params[:email]

    if email.blank?
      render text: "We cannot send your data without an email address. Please try again.", :status => :not_acceptable
      return
    end

    Exporter.perform_async(applet.report_id,params[:email])
    render text: "We are now exporting data from #{applet.name}. We will email #{email} when the report is ready."
  rescue StandardError => e
    Airbrake.notify(e)
    render text: "We're sorry, we had trouble exporting your data. Please try again.", status: 500
  end

  private

  attr_reader :applet

  def find_applet
    @applet = current_user.applets.find(params[:applet_id])
  end
end
```

## ..keep all classes thin/simple

### web/app/services/pusher_authorizer.rb

```ruby
require "ostruct"

class PusherAuthorizingResult < Struct.new(:success)
  def success?
    !!success
  end
end

class PusherAuthorizer
  def authorize(user,channel)
    @result = PusherAuthorizingResult.new

    topic, id = channel.scan(/private-(.*)-(\d+)/).flatten

    @result.success = case topic
      when "user"
        user.id.to_s == id
      when "connection"
        user.connections.exists?(id)
      else
        false
      end

    return @result
  end
end
```

## View templates should be very, very dumb

* https://github.com/objects-on-rails/display-case

## Be profligate with creating classes

## Use custom exception classes

```ruby
Staq::SecurityViolation = Class.new(StandardError)
Staq::Defect = Class.new(StandardError)
```

## Write copious documentation

* http://rubydoc.info/gems/yard/file/docs/Tags.md
```ruby
  ## Lets you emit a single datapoint from within an extractor parse.rb file,
  ## for storage in the database. Think of a single datapoint as a collection of
  ## facts, defined by 'values' about some particular, unique entity, defined by
  ## 'key', for an interval of time, defined by 'start_at' and 'end_at'
  #
  ## @example
  ##   emit do
  ##     key [agency_id,campaign_id] ## some combination of fields that makes this datapoint unique
  ##     start_at ref_time.beginning_of_hour ## does this datapoint cover an hour, a day, a minute?
  ##     end_at ref_time.end_of_hour
  ##     values(campaign_id: "500", impressions: 14010)
  ##   end
  #
  ## @note key can be a string or an array. If an array,  elements will be joined into a pipe-separated string.
  ##   If your key is greater than 255 characters, we will turn it into a SHA1 hash to guarantee uniquess but
  ##   stay within Redshift's column width constraints.
```

## Prefer composition over inheritance

```ruby
require_relative "collector_behavior"
require "sidekiq-pro"

class Collector
  include CollectorBehavior
  include Sidekiq::Worker

  sidekiq_options :queue => :collector, :retry => 5
end
```

## It isn't done unless it's

* Tested
* Documented
* Instrumented
* Shipped

## Minimal techno == maximal productivity
[Pandora Minimal Techno station](http://www.pandora.com/station/0407f1bebf01e992607b771623f63b252d6c8984f7343ea9)

## Questions?

> Everything should be made as simple as possible, but no simpler.
> - Albert Einstein

* mike@subelsky.com
* http://tinyletter.com/subelsky
