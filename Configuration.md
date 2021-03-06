- [Karafka configuration options](#karafka-configuration-options)
- [Connection pool configuration options](#connection-pool-configuration-options)
- [Ruby-Kafka driver configuration options](#ruby-kafka-driver-configuration-options)
- [External components configurators](#external-components-configurators)
- [Environment variables settings](#environment-variables-settings)
- [Kafka brokers auto-discovery](#kafka-brokers-auto-discovery)

Karafka is a complex tool, that contains multiple configuration options. To keep everything organized, all the configuration options were divided into three groups:

* Karafka options - options directly related to Karafka framework and it's components
* Connection pool options - options related to messages producers connection pool
* Ruby-Kafka driver options - options related to [Ruby-Kafka](https://github.com/zendesk/ruby-kafka)

To apply all those configuration options, you need to use the ```#setup``` method from the Karafka::App class (karafka.rb):

```ruby
class App < Karafka::App
  setup do |config|
    config.client_id = 'my_application'
    config.backend = :inline
    config.batch_fetching = true
    config.batch_consuming = true
    config.kafka.seed_brokers = %w[kafka://127.0.0.1:9092]
  end
end
```

Note: Karafka allows you to redefine most of the settings per each consumer group, which means that you can have specific custom configuration, that might differ from the default one configured on the app level. This allows you for example, to connect to multiple Kafka clusters.

## Karafka configuration options

| Option            | Required | Value type   | Default                 | Description                                                                                           |
|-------------------|----------|--------------|-------------------------|-------------------------------------------------------------------------------------------------------|
| client_id         | true     | String       | -                       | Application name that will be used as a client_id for Kafka cluster                                   |
| backend           | false    | Symbol       | :inline                 | Backend for consumption that we want to use (currently :inline or :sidekiq)                           |
| logger            | false    | Object       | Logger                  | Logger instance                                                                                       |
| monitor           | false    | Object       | Monitor                 | Monitor instance                                                                                      |
| consumer_mapper   | false    | Module       | Routing::ConsumerMapper | Mapper for building consumer ids                                                                      |
| topic_mapper      | false    | Module       | Routing::TopicMapper    | Mapper for hiding Kafka provider specific topic prefixes/postfixes, so internaly we use "pure" topics |
| batch_fetching    | false    | Boolean      | true                    | Should the incoming messages be fetched in batches, or one at a time                                  |
| batch_consuming   | false    | Boolean      | false                   | Should the incoming messages be consumed/processed in batches, or one at a time                       |
| shutdown_timeout  | false    | Integer, nil | 60                      | The number of seconds after which Karafka longer waits for the consumers to stop gracefully           |
| deserializer      | false    | Object       | Karafka::Serialization::Json::Deserializer | Default deserializer that will be used to deserialize incoming data                |
| serializer        | false    | Object       | Karafka::Serialization::Json::Serializer   | Default serializer that will be used to serialize outgoing data                    |

### Ruby-Kafka driver configuration options

We've listed here only **the most important** configuration options. If you're interested in all the options, please go to the [config.rb](https://github.com/karafka/karafka/blob/master/lib/karafka/setup/config.rb) file for more details.

| Option                         | Required | Value type    | Default | Description                                                                                       |
|--------------------------------|----------|---------------|---------|---------------------------------------------------------------------------------------------------|
| kafka.seed_brokers             | true     | Array<String> | -       | Kafka server hosts. Karafka will discover whole cluster structure automatically                   |
| kafka.start_from_beginning     | false    | Boolean       | true    | Consume messages starting at the beginning or consume new messages that are produced at first run |
| kafka.offset_commit_interval   | false    | Integer       | 10      | The interval between offset commits in seconds                                                    |
| kafka.offset_commit_threshold  | false    | Integer       | 0       | The number of messages that can be consume before their offsets are committed                     |
| kafka.heartbeat_interval       | false    | Integer       | 10      | The interval between heartbeats                                                                   |
| kafka.ssl_ca_cert              | false    | String        | nil     | SSL CA certificate                                                                                |
| kafka.ssl_client_cert          | false    | String        | nil     | SSL client certificate                                                                            |
| kafka.ssl_client_cert_key      | false    | String        | nil     | SSL client certificate password                                                                   |
| kafka.ssl_ca_certs_from_system | false    | Boolean       | false   | Use the CA certs from your system's default                                                       |

## External components configurators

For additional setup and/or configuration tasks you can use the `app.initialized` event hook. It is executed **once** per process, right after all the framework components are ready (including those dynamically built). It can be used for example to configure some external components that need to be based on Karafka internal settings.

```ruby
class App < Karafka::App
  # Setup and other things...

  # Once everything is loaded and done, assign Karafka app logger as a Sidekiq logger
  # @note This example does not use config details, but you can use all the config values
  #   to setup your external components
  monitor.subscribe('app.initialized') do
    Sidekiq::Logging.logger = Karafka::App.logger
  end
end

# Or if you prefer, you can do it from the outside of the app

Karafka.monitor.subscribe('app.initialized') do
  Sidekiq::Logging.logger = Karafka::App.logger
end
```

## Environment variables settings

There are several env settings you can use:

| ENV name          | Default | Description                                                                           |
|-------------------|-----------------|-------------------------------------------------------------------------------|
| KARAFKA_ENV       | development     | In what mode this application should boot (production/development/test/etc)   |
| KARAFKA_BOOT_FILE | app_root/karafka.rb | Path to a file that contains Karafka app configuration and booting procedures |

## Kafka brokers auto-discovery

Karafka supports Kafka brokers auto-discovery during startup and on failures. You need to provide at least one Kafka broker, from which the entire Kafka cluster will be discovered. Karafka will refresh list of available brokers if something goes wrong. This allows it to be aware of changes that happen in the infrastructure (adding and removing nodes).