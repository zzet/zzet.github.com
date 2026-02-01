---
layout: post
title:  "Project configuration. Part 1. Railsclub intro."
date:   2017-10-29
desc: "Intro into project configuration thread"
keywords: "ruby,ruby on rails,configuration,monilith,microservice"
tags: [configuration,conference,railsclub 2017]
icon: icon-microscope

uglyURLs: true
aliases:
- /2017/10/29/project-configuration-railsclub-intro
- /2017/10/29/project-configuration-railsclub-intro.html
---

Смените язык внизу станицы для доступа к оригинальной статье или перейдите по [ссылке]({{% ref path="/archive/2017-10-29-project-configuration-railsclub-intro" lang="ru" %}})

This article was originally a transcript of a talk given at the Railsclub 2017 conference. Minor corrections have been made to the text following the conference.

In the process of reading, you will understand why it makes no sense to describe every detail immediately. I do not absolve myself of responsibility for providing the information of interest. You can write in the comments what questions have arisen and what should be explained in more detail. This will serve as a trigger for detailed publications.

Let's go.

Today I want to talk about a topic that might seem old, hackneyed, and uninteresting to those around us. But actually, that's not the case. As they say, everything new is well-forgotten old, especially if the old **needs** to be updated. I think you've heard that all talks are divided into 2 types: bragging or confessing. Today I will be doing the second (probably).

While preparing for the talk, I thought about the best way to start the narrative... I came up with a tearful story about my experience fighting castles in the air, found some scary pictures... And then I went to Wikipedia and... found the answer to the question I want to cover.

> Software configuration is the collection of program settings defined by the user, as well as the process of changing these settings in accordance with the user's needs.
> 

The problem is that the majority only looks at the first part of this definition, and (very often) do not take the second part into account. Of course, the problem isn't just that—we will look at key aspects—but this specific question prompted me to reconsider my views on project configuration and develop a new approach. And the question is not just about how to update configuration in the literal sense. No, I won't talk about how you need to press keys to generate code. I want to touch upon the problem of introducing changes to configuration, delivering these changes to production, and working in different environments with the same code. That is what I will talk to you about today. And yes, the conversation will be in the context of Ruby, of course (but it can be translated to other languages and technologies) :)

I'll start with a small lyrical digression. Many people here say that Ruby is dead, that it isn't developing, that the world is decay, and it's time to bail. It just so happened that over the last 3 years, I've seen a number of completely different projects. Different not only in terms of domain but also in terms of architecture. There were monoliths, semi-monoliths, semi-semi-monoliths, and projects based on microservice architecture. And projects on MRI, on JRuby, and mixtures with other languages. A zoo, in a word.

So, in each of these projects, different approaches to the configuration of these projects were applied. The approaches were different, and the problems were also different. But all these *problems* boiled down to the same word: **configuration**.

It happened that while participating in one project, I simply started to—as they say—burn out. The competence and professionalism of the guys on the team raised no questions, but the approaches and certain decisions... for a long time, I couldn't understand why exactly these decisions were made, but I saw that they were leading nowhere. And then I realized. The problem was on the surface: **We preach what we are used to**. Therefore, I will consider the issue of **Habit** first and foremost.

When we spoke with the Railsclub conference organizers about this talk, I sent them this [video](https://www.youtube.com/watch?v=FeCXooh8AXE). Enough time has passed since then for me to find politically correct words, systematize my hatred, and calmly lay everything out.

# Convention over Configuration

The funniest thing in this story is that the Rails ecosystem is built on conventions, yet the question of configuration specifically does not obey this rule, and in this matter, it's a free-for-all. Yes, it's just such a paradox. We have conventions so there isn't a ton of configuration, but you still can't escape configuration.

If anyone doesn't know, Convention over Configuration (CoC) is a design principle for frameworks and libraries intended to reduce the amount of required configuration without losing flexibility. In a strict form, this principle can be expressed as follows: an aspect of a software system needs configuration if and only if that aspect DOES NOT satisfy a certain specification. The principle works when it comes to mapping classes to certain resources (database tables, events, file system resources). According to the principle, if a class matches the naming convention, then it does not need additional configuration. In this context, the name of the principle indicates the primacy of the convention over the configuration.

A classic example of the CoC principle is Hibernate. In Hibernate, object-relational mapping rules can be described using XML files:

XML

# 

`<class name="Tag" table="tag">
    <property name="Name" column="name"/>
    <property name="Value" column="value"/>
    <property name="Value2" column="value2"/>
    <property name="Value3" column="value3"/>
</class>`

As you can see, there are repetitions of class properties and table columns. If we introduce a convention that by default table columns must be named the same as the property, we can omit part of the configuration:

XML

# 

`<class name="Tag">
    <property name="Name"/>
    <property name="Value"/>
    <property name="Value2"/>
    <property name="Value3"/>
</class>`

However, if some property needs to be saved in a column with a different name, this will have to be specified explicitly. This is the essence of the principle.

The CoC principle finds its greatest application in the Ruby on Rails environment. This is understandable in principle, given that RoR is oriented towards rapid development, and CoC allows configuration to be reduced to a minimum.

And essentially, this is correct, but! But this in no way removes responsibility from configuration issues. It is incorrect to think that all problems have already been solved for you by other people, and if something in your work goes beyond the guides, then you can write "a little bit of garbage code."

# Monolith - Not a Monolith

Although, I started off on the wrong foot. First, I should ask, who among you works on a monolith project?

For these people, most likely, the first part of the article is not interesting, because everything is much simpler (no, not simple, there are problems) when you have a monolithic application. What do we have: all changes are stored in one place, they are much easier to synchronize. Very often, no *serious* problems arise with monoliths. And if conflicts do arise in terms of configuration—they don't cause much pain. A few abstract crutches, a little terror towards DevOps, and that's it, most problems are solved. But when it starts to split into small services (I advise not to rush into this if you don't know what you are getting into)—here many questions arise regarding how to maintain consistency, including in configuration. After all, everyone understands perfectly well that the more independent components there are in a system, the harder they are to synchronize. And then the rule "if it works, don't touch it" kicks in, which we will talk about later :).

Therefore, I have several goals today:

1. To grumble about how people shoot themselves in the foot and consider it normal.
2. To sow a note of doubt among some people regarding the fact that everything is fine and nothing needs to be done/changed.
3. To shed light on the problem of microservices in Ruby (and not only in Ruby, but we are at Railsclub, so Ruby).

# Existing Approaches

And for this, we need to look at a retrospective; there's no way around it. Does everyone remember how approaches to configuration evolved? By the way, did everyone notice that I haven't asked anyone yet what they consider to be *configuration*?

## Constants

The simplest, fastest, and in some cases, indispensable option. Just recall how a ruby gem version is described.

Ruby

# 

`# lib/persey/version.rb
module Persey
  VERSION = "0.0.11"
end

# persey.gemspec
# ...
Gem::Specification.new do |spec|
  # ...
  spec.version = Persey::VERSION
  # ...
end`

I see the approach using constants as a tool for configuration very often, especially in various libraries.

Ruby

# 

`# https://github.com/rails/rails/blob/master/actioncable/lib/action_cable/gem_version.rb
module ActionCable
  def self.gem_version
    Gem::Version.new VERSION::STRING
  end

  module VERSION
    MAJOR = 5
    MINOR = 2
    TINY  = 0
    PRE   = "alpha"

    STRING = [MAJOR, MINOR, TINY, PRE].compact.join(".")
  end
end`

Or this:

Ruby

# 

`# https://github.com/rails/rails/blob/master/actioncable/lib/action_cable.rb
module ActionCable
  extend ActiveSupport::Autoload

  INTERNAL = {
    message_types: {
      welcome: "welcome".freeze,
      ping: "ping".freeze,
      confirmation: "confirm_subscription".freeze,
      rejection: "reject_subscription".freeze
    },
    default_mount_path: "/cable".freeze,
    protocols: [
      "actioncable-v1-json".freeze,
      "actioncable-unsupported".freeze
    ].freeze
  }
  # ...
end

# https://github.com/rails/rails/blob/master/actioncable/lib/action_cable/connection/web_socket.rb

module ActionCable
  module Connection
    # Wrap the real socket to minimize the externally-presented API
    class WebSocket # :nodoc:
      def initialize(env,
                     event_target,
                     event_loop,
                     protocols: ActionCable::INTERNAL[:protocols]
                    )
         # ...
      end
      # ...
    end
  end
end`

Projects of a larger scale are no exception (unfortunately). And if, in the examples above, this can somehow be called constants, in reality, developers get so carried away that they write whatever they want into constants.

Once I was at an interview at a company. During the interview, we touched on the topic "What do you think, can constants be used to store configuration?". I answered unequivocally no, that it is bad. Spoiler (this is exactly why my candidacy was rejected, for which I thank them very much).

What arguments were given in favor of using constants?

1. Constants can be accessed from everywhere (if the file with constants is loaded).
2. Constants help improve code readability. Magic numbers and strings disappear.
3. Easy to search through the code (you search for the constant name, not a magic number).
4. Cannot be modified at runtime.

Well, in principle, everything is true (almost). But there are counterarguments too.

1. No one keeps constants in one place. They are always smeared across the code. To reuse a particular constant, you need to know where it lies (in which file/module/class). And no one wants to memorize this (moreover, no one needs to). As a result, everything leads to developers starting to duplicate these configuration parameters. As soon as you need to change a particular magic string, the game of "guess what the colleague named the constant" begins in order to find it in the code. Is this problem curable? Yes, more discipline, concentration, and attention. But why all this?
2. The line between a constant value and a configuration parameter begins to blur. People start stuffing everything that comes to mind into constants.

Ruby

# 

`RETRIES_COUNT = 10
API_HOST = 'https://your.domain.com'`

1. Different environments turn life into hell. In the test environment one value is needed, on staging another, in production a third, and the project might not be limited to three or four environments.

Ruby

# 

`RETRIES_COUNT = case Rails.env
                when 'test'
                  2
                when 'development'
                  5
                when 'staging'
                  10
                else
                  50
                end

API_HOST = case Rails.env
                when 'test'
                  'https://fake.domain.com'
                when 'development'
                  'https://loalhost:3000'
                when 'staging'
                  'https://your-staging.domain.com'
                else
                  'https://your-production.domain.com'
                end`

1. Do not change? Yeah, right.

Ruby

# 

`class SomeClass
  CONSTANT_STRING = "string"
  CONSTANT_NUMBER = 1
  CONSTANT_ARRAY = [1, 2, 3]
  CONSTANT_HASH = { a: :b }
  CONSTANT_HELL = {
    a: CONSTANT_HASH,
    b: CONSTANT_ARRAY,
    c: {
      d: :e,
      f: [1, 2, 3]
    }
  }
end

[1] pry(main)> SomeClass::CONSTANT_STRING
=> "string"
[2] pry(main)> SomeClass::CONSTANT_STRING << 'smth'
=> "stringsmth"
[3] pry(main)> SomeClass::CONSTANT_STRING
=> "stringsmth"`

Everyone knows that everything is simple here, use `.freeze` and then no values can be changed.

Ruby

# 

`class SomeClass
  CONSTANT_STRING = "string".freeze
  CONSTANT_NUMBER = 1.freeze
  CONSTANT_ARRAY = [1, 2, 3].freeze
  CONSTANT_HASH = { a: :b }.freeze
  CONSTANT_HELL = {
    a: CONSTANT_HASH,
    b: CONSTANT_ARRAY,
    c: {
      d: :e,
      f: [1, 2, 3]
    }
  }.freeze
end

[1] pry(main)> SomeClass::CONSTANT_STRING
=> "string"
[2] pry(main)> SomeClass::CONSTANT_STRING << 'smth'
RuntimeError: can't modify frozen String

[3] pry(main)> SomeClass::CONSTANT_HELL[:a] = 123
RuntimeError: can't modify frozen Hash

[4] pry(main)> SomeClass::CONSTANT_HELL
=> {:a=>{:a=>:b}, :b=>[1, 2, 3], :c=>{:d=>:e, :f=>[1, 2, 3]}}
[5] pry(main)> SomeClass::CONSTANT_HELL[:c][:d] = 123
=> 123
[6] pry(main)> SomeClass::CONSTANT_HELL
=> {:a=>{:a=>:b}, :b=>[1, 2, 3], :c=>{:d=>123, :f=>[1, 2, 3]}}`

Except that freeze does not "freeze" nested objects, and one can very easily run into such a situation.

But credit where credit is due: redefining a constant won't work. This is perhaps the only thing they are good for.

If anyone is interested in asking about how this works inside the Ruby machine—we'll talk about it on the sidelines, so let's move on.

Yes, if someone wants to say that environment variables can be used at the same time—yes, they can. But that also doesn't work :)

## Config file

The second most popular approach is to put all parameters into YAML files, which will then be read and the resulting hash will be used as the configuration storage object.

There are undoubtedly pluses to this approach. As well as minuses, for that matter...

### Pros:

1. Finally, configuration has started accumulating in one place.
2. YAML files allow you to elegantly describe configuration, separate by environments, remove duplication.

YAML

# 

`common: &common
  adapter: postgresql
  host:     <%= ENV.fetch('DB_HOST') %>
  database: <%= ENV.fetch('DB_NAME') %>
  username: <%= ENV.fetch('DB_USERNAME') %>
  password: <%= ENV.fetch('DB_PASSWORD') %>
  encoding: unicode
  pool: <%= ENV.fetch('DB_POOL') %>

development:
  <<: *common

test:
  <<: *common

staging:
  <<: *common
  port: 5432`

1. You can use environment variables (the option of running the read file through the ERB preprocessor is used universally).
2. Separation of configs by domain areas is very good. I love it when everything is on the shelves.

### Cons:

1. You depend on the internal implementation of a particular gem (especially regarding the use of environment variables). For example, the database config goes through ERB preprocessing. But what happened to `cable.yml`? Why was it deprived?
2. You cannot re-apply the same configuration parameter between different files, as they are autonomous. An exception, of course, can be said right away—sharing these data through environment variables. Who among you has done this in a large project where there are quite a lot of such environment variables?

Bash

# 

  `$ grep -R ENV config app lib | wc -l
      458`

1. You cannot reuse parts of the config within the config itself (well, strictly speaking, this isn't true, what is meant is that you can't do it elegantly).
2. It is difficult to maintain changes if the file structure changes.
3. And the biggest problem—it is difficult to track changes in these configuration files.

I encountered the last point quite a long time ago when contributing to Gitlab. There were 3 main problems:

1. Synchronization of file changes (source files were supplied as `config.yml.example`), which had to be copied and rewritten. As soon as new parameters appear in `config.yml.example` that you, due to some circumstances, did not notice, problems can begin.
2. Support for changes in the files themselves (backward compatibility support).
3. Duplication of configuration parameters (there was gitlab-shell and gitlab itself. Both parts had to know which path the repositories lay on. If you had to change one parameter—you had to remember the other. But wait, this is one product, what is this nonsense?)

How were these problems solved? They weren't. Well, we solved them internally ;), but `Undev` is no more and no one remembers that anymore. I, at least, hope so.

Well, at the moment, this approach leads in the community of rails (and not only) developers.

## DSL

This variant should probably have been mentioned earlier, as it appeared and took root earlier than the variant with configuration files. But I put it as the next item because it can still be used in a more flexible variant.

I think everyone knows about

Ruby

# 

`Rails.application.configure do
# ...
end`

So this variant is exactly from this opera.

Pros:

1. The config reads very well.
2. Config == code. Possibilities for describing configuration increase.
3. Can be tested (!!!). Yes, the config can be tested and one can be sure that everything is fine with it. Except nobody does this. Here, of course, one can argue about what needs to be tested and what does not.
4. All the pros from the **Config object**.

I particularly like the `Dry-configurable` variant (I am generally a fan of Dry).

Ruby

# 

`class App
  extend Dry::Configurable

  # Pass a block for nested configuration (works to any depth)
  setting :database do
    # Can pass a default value
    setting :dsn, 'sqlite:memory'
  end
  # Defaults to nil if no default value is given
  setting :adapter
  # Pre-process values
  setting(:path, 'test') { |value| Pathname(value) }
  # Passing the reader option as true will create attr_reader method for the class
  setting :pool, 5, reader: true
  # Passing the reader attributes works with nested configuration
  setting :uploader, reader: true do
    setting :bucket, 'dev'
  end
end

App.config.database.dsn
# => "sqlite:memory"

App.configure do |config|
  config.database.dsn = 'jdbc:sqlite:memory'
end

App.config.database.dsn
# => "jdbc:sqlite:memory"
App.config.adapter
# => nil
App.config.path
# => #<Pathname:test>
App.pool
# => 5
App.uploader.bucket
# => 'dev'`

## Singleton

Also, a similar variant of storing configuration has taken root quite firmly in the Ruby world (including in the Rails code itself).

The essence of this approach boils down to the following: configuration data is read from some source (sources) and saved in one object.

The pros and cons depend on where and how this data is taken.

## DB

I always remember the times working in a web studio, churning out sites on CMSs, when I see such an approach. When there is a table in the database, configuration parameters are stored there as key => value, and the developer runs to this database from the code to read certain values. Actually, there is nothing disgustingly bad about this. This is one way to share configuration that can change at any moment. And it doesn't matter where it will be (PostgreSQL, Redis, etc.).

Problems:

- Very fragile system (lives only if the key cannot be deleted).
- Load on the database can be very high.
- Parasitic traffic.
- Drops in application speed.

Pros:

- Configuration of individual system components can be changed on the fly.
- Sharing configuration between different services out of the box.

# Problems that the majority ignore

## Many configs

Obviously, the more complex the application, the more libraries are used. And often, each library asks for its own configuration file. Speaking in the context of Rails, this leads to 2 directories starting to swell from the number of such files:

`config`

`config/initializers`

The problem here, as you understand, is not in the number of files, but in the number of variables/modules/classes that store configuration values.

One way or another, the growth in the number of such entities leads to the appearance of additional abstractions in order to observe DRY principles, which leads to high code coupling and less understanding of what is happening. And an additional problem begins to manifest: how to ensure configuration consistency. The problem of maintenance. However, this problem is easily shifted onto the shoulders of other people (for example, DevOps, we talked about how all these problems can be solved using environment variables, although this is also nonsense, at least because eventually, it becomes impossible to invent and combine a large number of variables).

## Different file formats

Not so many teams and projects encounter such a situation, but it happens and it is worth mentioning. How many people use in their solutions not only ruby libraries but specific software that has a very concrete configuration?

What if there are small services that have their own configs? You still need (not always) to take into account parameters from these configs. Copy-paste? Pass through environment variables? And what to do if the service is launched by a supervisor at system startup? Juggle environment variables? And if the service cannot read these environment variables (can only read from a file)? Rewrite the service just so it can do this? And if you don't know the programming language or the software is proprietary?

All these questions ultimately boil down to the fact that you just need to duplicate important parameters. Just duplicate, instead of reusing. Yes, I do not argue, this question can also be solved with the help of DevOps. And even then, not always. In general, sometimes a situation arises where you just need to read one more config and have access to it from the application code.

## Dynamic configuration & using configuration within configuration

How often do you need to assemble 1 parameter from several configuration parameters?

You don't need to look far—the database connection string.

Let's recall the database config.

YAML

# 

`common: &common
  adapter: postgresql
  host:     <%= ENV.fetch('DB_HOST') %>
  database: <%= ENV.fetch('DB_NAME') %>
  username: <%= ENV.fetch('DB_USERNAME') %>
  password: <%= ENV.fetch('DB_PASSWORD') %>
  encoding: unicode
  pool: <%= ENV.fetch('DB_POOL') %>

development:
  <<: *common

test:
  <<: *common

staging:
  <<: *common
  port: 5432`

And in the application, you need to use not only data in this format but also the connection string as a single string. What to do in this case?

YAML

# 

`common: &common
  adapter: postgresql
  host:     <%= ENV.fetch('DB_HOST') %>
  database: <%= ENV.fetch('DB_NAME') %>
  username: <%= ENV.fetch('DB_NAME') %>
  password: <%= ENV.fetch('DB_PASSWORD') %>
  encoding: unicode
  pool: <%= ENV.fetch('DB_POOL') %>
  dsn: postgres://<%= ENV.fetch('DB_NAME') %>:<%= ENV.fetch('DB_PASSWORD') %>@<%= ENV.fetch('DB_HOST') %>:<%= ENV.fetch('DB_PORT') %>/<%= ENV.fetch('DB_NAME') %>?pool=<%= ENV.fetch('DB_POOL') %>`

Does no one see a problem with this? Actually, there is no problem, there is an annoying nuisance in the form of the need to control 2 places where the environment variable is used instead of one.

But what if it's the other way around, extracting several attributes from 1 parameter? You have a `dsn` at the input and need to break it into separate parameters (for example, the connection string to AWS RDS DB)?

Ruby

# 

`common: &common
  dsn:      <%= ENV.fetch(‘DB_DSN’) %>
  adapter:  ???
  host:     ???
  database: ???
  username: ???
  password: ???
  encoding: unicode
  pool: ???`

Or supplement 1 parameter with additional data (for example, several databases in Redis)? Here, the hemorrhoid increases a bit more.

I don't particularly see the point in clogging the environment variable scope for every sneeze. It is enough to demand support for such behavior/features from the part of the project responsible for configuration.

## Combining different types of configuration

And what to do when you have configuration described using DSL, configuration lying in the database, files with configs, configs passed through environment variables? Convert everything to one format? For example, use only YAML files or only DSL? Actually, this is a correct thought and I would choose the second option. At least in this case, you can use `lambda` to include one config in the second.

## Reusing configuration in parent projects (config registry)

This is also an interesting moment. Let's imagine a situation: you are writing a small application that takes data from an API, does something with it, writes the result to a file, and uploads it to an AWS S3 bucket. In this abstract task, 3 components can be distinguished:

1. Call to API.
2. Performing the operation.
3. Uploading the file.

Obviously, each of these parts (at least 2) will require credentials. And most likely, they will lie in 2 files side by side and the first and third components will read them perfectly and everything will work. Now the question: what is the probability that in the second component you will need to take into account configuration parameters from 1 and 3?

Here the following questions arise:

1. Are you using a third-party solution?
2. Access the configuration object?
3. What if the configuration is encapsulated and there is no public access to it?

Of course, this is Ruby, you can call private methods and reach the most spicy places, risking that your code will break when the internal interface of the library changes.

Or it is simpler to combine these 2 configs (make a proxy interface to them).

And now let's imagine that we have some description of models that your services can work with, and you extracted them into a separate gem. One way or another, this gem will have configuration, and you may need access to the parameters of some keys from your service. How to proceed in this case? Duplicate configs? Or reuse the configuration object from the parent class?

And if you connected another 1 library, and it has its own config? Access this config directly or also pass access to it through your config? If you choose the first option, doesn't it seem to you that this is similar to the story with smearing constants? And what about the rule of one level of abstraction? And how to guarantee that your configuration interface will be immutable (replacing the backend does not call for changing the code)?

## Reloading configs

### App restart vs configuration reload

Let's return briefly to monoliths. When you have an application executed as a monolith, it has both pros and cons in terms of the ability to use such an approach to configuration updates. It is worth starting with the fact that there is no particular practical sense in reloading configuration without restarting the application. At least because changes in the project itself (usually) happen quite quickly and the probability that a configuration change might come with a code change is very high. And changing configuration in this case is equivalent to changing code, due to the fact that code coupling is very high.

The situation changes slightly when the monolith is launched not quite as a monolith. Despite the fact that the code lies in one place, all these components are launched separately. For example, the API separately (and the API itself can also be separated, for example by versions), the web part of the personal account separately, public controllers separately, background tasks of `critical` level separately, and so on.

And everything changes radically when instead of 1 large application you get many small ones (or as people like to call it - microservices).

In the last 2 cases, the question becomes legitimate:

**Does it make practical sense to restart all services if the code in them has not changed, but changes have occurred in the configuration?**

Of course, this question has many other parameters:

- What is the cost of restarting the service?
- Is it necessary to restart the service?

Obviously, the smaller the service, the lighter its restart. And the downtime here can be quite small. But what does this restart represent? **Graceful restart** or **SIGKILL** + **SIGEXIT**? If you have the second option—restarting all services when necessary to reload configuration becomes a dangerous venture. At least because some service might:

1. Be performing heavy operations.
2. Contain a cache that needs to be warmed up after restart.
3. Premature termination of work may affect data consistency.
4. Very frequent restarts can block the flow.

These points alone are enough to doubt the usefulness of restarting all services when changing one.

The desire to keep all services running "on the freshest code" is inherent in many developers. Sometimes to the point of fanaticism. Hmm. Do all of you update gems every day (to guarantee that the freshest version of the gem is used)? Then why restart the service if no changes have occurred in it (or changes in a neighboring service do not affect changes in a specific one)?

Thus, for me, the priority question is: **Do we need to restart it?** and **Do changes in configuration/code affect the work of the application itself?**

Let's analyze the second moment in more detail:

It is considered good practice to have isolated storage for each microservice. By storage, one can understand, for example, a database. And all data is passed between these services over the network. But in practice, this is practically unattainable (at least immediately). One way or another, at the very beginning you will have, for example, a database shared between these services. Not necessarily between all of them. Or you need to use separate storage for communications (use the result of one service's work in another, while the volume of data can be quite large (We had a case where several million rows from one service needed to be transferred to another). In such a case, it turns out that a change, for example, in part of the configuration (information about the location of such storage) does not affect all system services, but only a limited number of these services, correspondingly, these changes should not affect others.

In that case, a possibility is needed to inform the application that the configuration is outdated and needs to be updated. How to do this correctly?

### API interface for updating configuration

For some reason, it is most often proposed to implement an API on the side of each service that can be knocked on to report that some change has occurred. Well... So-so solution. It feels kinda lazy to **implement** such an interface for each service independently.

### SIGHUP to support configuration reload

The idea boils down to sending a **SIGHUP** signal to the service so that it updates the configuration (re-reads it). Such an approach still has wide application in the Ruby world. But this is logical, as everyone is used to restarting the application if something has changed.

### Configuration conflict problem (configuration change at runtime)

Let's imagine such a situation: We know how to reload configuration in real-time. In such a case, we can get a situation where the code was launched on one configuration, an update occurred, and it started working based on another configuration. On one hand, this might not be scary, on the other... it might be useful.

For example, you were writing part of the data to one place, then information arrived that you need to write to another place and you decided to switch. This looks somewhat not very good. Part of the data in one place, part of the data in another, and it is unknown where and how much.

Or you had a limit on string length. It was `1000` characters, it became `800`. Part of the generated strings will correspond to the required result, part will not. Is this good or bad? I cannot say unequivocally. For us, similar situations were both useful and harmful. But it is better when the result corresponds to a specific criterion. Either all `1000` characters, or all `800`.

Accordingly, it is better to keep this under control somehow. How many people use the `config object` pattern in their work?

## Using the config object approach

We pass a configuration object into the class/method and use this snapshot. During the operation execution, the config may change, the snapshot remains unchanged. If the configuration changes were critical—execution will fail and the service will recover with the new configuration, continue work, and everything will be fine.

Ruby

# 

`def generate_ad_headline(headline_parts: [])
  headline = headline_parts.each_pair.each_with_object({}) do |(ad_id, parts), acc|
    acc[ad_id] = parts.each_with_object("") do |part, result_headline|
      if (result_headline + part).size < YourApp.config.ads.headline.max_size
        result_headline << part
      end
    end
  end
end

def generate_ad_headline(headline_parts: [], config_object: YourApp.config.ads.headline)
  headline = headline_parts.each_pair.each_with_object({}) do |(ad_id, parts), acc|
    acc[ad_id] = parts.each_with_object("") do |part, result_headline|
      if (result_headline + part).size < config.max_size
        result_headline << part
      end
    end
  end
end`

## pub/sub configuration update model

Consul config

etcd

similar services

Problem: you have several services

# Project configuration is not just magic strings

Flexible code with a minimal set of changes.

## Service locators as part of configuration (DI)

Do not tie yourself to a specific service (gem interface) or class communicating with the outside world directly. Use service locators.

**Implementation example using Dry-container**

Ruby

# 

`class ServiceLocator
  class Container
    include Dry::Container::Mixin
  end

  class << self
    attr_reader :instance

    def configure
      container = Container.new
      yield(container)
      @instance = new(Rails.application, container)
      freeze
    end

    def [](name)
      instance[name]
    end
  end

  attr_reader :app, :container

  def initialize(app, container)
    @app = app
    @container = container
  end

  def [](name)
    container[name]
  end
end

ServiceLocator.configure do |container|
  load_environment = begin
                        if ENV['PRODUCTION_SERVICE_LOCATOR']
                          :prod
                        elsif Rails.env.test? || Rails.env.development?
                          :fake
                        else
                          :prod
                        end
                      end

  case load_environment
  when :fake
    container.register(:sms_gateway, lambda do
      Test::Nexmo::Client.new(
        key: PactApi.config.nexmo.api_key,
        secret: PactApi.config.nexmo.api_secret
      )
    end)

    container.register(:mailgun_client, lambda do
      Test::Mailgun::Client.new(PactApi.config.mailgun.api_key)
    end)
  else
    container.register(:sms_gateway, lambda do
      Nexmo::Client.new(
        key: PactApi.config.nexmo.api_key,
        secret: PactApi.config.nexmo.api_secret
      )
    end)

    container.register(:mailgun_client, lambda do
      Mailgun::Client.new(PactApi.config.mailgun.api_key)
    end)
  end
end

email_message.take_to_send!

mg_client = ServiceLocator[:mailgun_client]
begin
  response = mg_client.send_message PactApi.config.mailgun.domain, payload

  if response.code == 200
    mailgun_message_id = response.to_h['id']
    mailgun_message_id.gsub!(/[<>]/, '')
    email_message.update(external_id: mailgun_message_id)
    email_message.sent!
  else
    email_message.fail_send!
  end
rescue
  email_message.fail_send!
end`

## Rail-way coding

When implementing, try to explicitly implement the true-way process.

# Conclusion