# apm.cr

This is an [APM](https://www.therelicans.com/wyhaines/the-basicest-basics-of-how-an-apm-system-works-847) agent implementation for Crystal.It is being built with OpenTelementry protocol support, which should enable the agent to deliver metrics, traces and logs to multiple vendors, such as [New Relic](https://trynewrelic.com), Datadog, and others.

It will hook into an application, automatically, to enable collection and reporting of various metrics relating to performance and behavior.

## Installation

1. Add the dependency to your `shard.yml`:

   ```yaml
   dependencies:
     apm:
       github: your-github-user/apm
   ```

2. Run `shards install`

## Usage

```crystal
require "apm"
```

TODO: Write usage instructions here

## Development

TODO: Write development instructions here

## Contributing

1. Fork it (<https://github.com/your-github-user/apm/fork>)
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create a new Pull Request

## Contributors

- [Kirk Haines](https://github.com/your-github-user) - creator and maintainer
