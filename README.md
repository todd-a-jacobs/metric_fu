# MetricFu [![Gem Version](https://badge.fury.io/rb/metric_fu.svg)](http://badge.fury.io/rb/metric_fu) [![Test](https://github.com/metricfu/metric_fu/actions/workflows/ruby.yml/badge.svg)](https://github.com/metricfu/metric_fu/actions/workflows/ruby.yml) [![Join the chat at https://gitter.im/metricfu/metric_fu](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/metricfu/metric_fu?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge) [![Code Climate](https://codeclimate.com/github/metricfu/metric_fu.svg)](https://codeclimate.com/github/metricfu/metric_fu) [![Inline docs](http://inch-ci.org/github/metricfu/metric_fu.svg)](http://inch-ci.org/github/metricfu/metric_fu)

[MetricFu](http://rdoc.info/github/metricfu/metric_fu/) is a set of tools to provide reports that show which parts of your code might need extra work.

Metrics supported: [Cane](http://github.com/square/cane), [Churn](http://github.com/danmayer/churn), [Flog](https://github.com/seattlerb/flog), [Flay](https://github.com/seattlerb/flay), [Reek](https://github.com/troessner/reek), [Roodi](https://github.com/roodi/roodi), [Saikuro](https://github.com/metricfu/Saikuro), [Code Statistics](https://github.com/bf4/code_metrics), [Hotspots](https://github.com/chiku/hotspots), [SimpleCov](https://github.com/simplecov-ruby/simplecov), [Rcov](https://github.com/relevance/rcov), [Rails Best Practices](https://github.com/flyerhzm/rails_best_practices) (Rails-only).

## Installation

Run:

```
$ gem install metric_fu
```

Or add this line to your application's Gemfile:

```ruby
gem 'metric_fu'
```

And then execute:

```
$ bundle
```

### Considerations

MetricFu is [cryptographically signed](http://guides.rubygems.org/security/).
To be sure the gem you install hasn't been tampered with:

- Add my public key (if you haven't already) as a trusted certificate `gem cert --add <(curl -Ls https://raw.github.com/metricfu/metric_fu/master/certs/bf4.pem)`
- `gem install metric_fu -P MediumSecurity`

The MediumSecurity trust profile will verify signed gems, but allow the installation of unsigned dependencies.

This is necessary because not all of MetricFu's dependencies are signed, so we cannot use HighSecurity.

## Usage

From your application root, run:

```
$ metric_fu
```

To see available options run `metric_fu --help`

## Configuration

MetricFu will attempt to load configuration data from a
`.metrics` file, if present in your repository root.

```ruby
MetricFu.report_name # by default, your directory base name
MetricFu.report_name = 'Something Convenient'
```

### Example Configuration

```ruby
# To configure individual metrics...
MetricFu::Configuration.run do |config|
  config.configure_metric(:cane) do |cane|
    cane.enabled = true
    cane.abc_max = 15
    cane.line_length = 80
    cane.no_doc = 'y'
    cane.no_readme = 'y'
  end
end

# Or, alternative format
MetricFu.configuration.configure_metric(:churn) do |churn|
  churn.enabled = true
  churn.ignore_files = 'HISTORY.md, TODO.md'
  churn.start_date = '6 months ago'
end

# Or, to (re)configure all metrics
MetricFu.configuration.configure_metrics.each do |metric|
  if [:churn, :flay, :flog].include?(metric.name)
    metric.enabled = true
  else
    metric.enabled = false
  end
end
```

### Rails Best Practices

```ruby
MetricFu::Configuration.run do |config|
  config.configure_metric(:rails_best_practices) do |rbp|
    rbp.silent = true
    rbp.exclude = ["config/chef"]
  end
end
```

### Coverage Metrics

```ruby
MetricFu::Configuration.run do |config|
  config.configure_metric(:rcov) do |rcov|
    rcov.coverage_file = MetricFu.run_path.join("coverage/rcov/rcov.txt")
    rcov.enable
    rcov.activate
  end
end
```

If you want metric_fu to actually run rcov itself (1.8 only), don't specify an external file to read from

#### Rcov metrics with Ruby 1.8

To generate the same metrics metric_fu has been generating run from the root of your project before running metric_fu

```shell
RAILS_ENV=test rcov $(ruby -e "puts Dir['{spec,test}/**/*_{spec,test}.rb'].join(' ')") --sort coverage --no-html --text-coverage --no-color --profile --exclude-only '.*' --include-file "\Aapp,\Alib" -Ispec > coverage/rcov/rcov.txt
```

#### Simplecov metrics with Ruby 1.9 and 2.0

Add to your Gemfile or otherwise install

```ruby
gem 'simplecov'
```

Modify your [spec_helper](https://github.com/metricfu/metric_fu/blob/master/spec/spec_helper.rb) as per the SimpleCov docs and run your tests before running metric_fu

```ruby
#in your spec_helper
require 'simplecov'
require 'metric_fu/metrics/rcov/simplecov_formatter'
SimpleCov.formatter = SimpleCov::Formatter::MetricFu
# or
SimpleCov.formatter = SimpleCov::Formatter::MultiFormatter[
  SimpleCov::Formatter::HTMLFormatter,
  SimpleCov::Formatter::MetricFu
  ]
SimpleCov.start
```

Additionally, the `coverage_file` path must be specified as above and must exist.

### Formatters

#### Built-in Formatters

By default, metric_fu will use the built-in html formatter to generate HTML reports for each metric with pretty graphs.

These reports are generated in metric_fu's output directory (`tmp/metric_fu/output`) by default. You can customize the output directory by specifying an out directory at the command line
using a relative path:

```sh
metric_fu --out custom_directory    # outputs to tmp/metric_fu/custom_directory
```

or a full path:

```sh
metric_fu --out $HOME/tmp/metrics      # outputs to $HOME/tmp/metrics
```

You can specify a different formatter at the command line by referencing a built-in formatter or providing the fully-qualified name of a custom formatter.

```sh
metric_fu --format yaml --out custom_report.yml
```

Or in Ruby, such as in your `.metrics`

```ruby
# Specify multiple formatters
# The second argument, the output file, is optional
MetricFu::Configuration.run do |config|
  config.configure_formatter(:html)
  config.configure_formatter(:yaml, "customreport.yml")
  config.configure_formatter(:yaml)
end
```

#### Custom Formatters

You can customize metric_fu's output format with a custom formatter.

To create a custom formatter, you simply need to create a class
that takes an options hash and responds to one or more notifications:

```ruby
class MyCustomFormatter
  def initialize(opts={}); end    # metric_fu will pass in an output param if provided.

  # Should include one or more of...
  def start; end           # Sent before metric_fu starts metric measurements.
  def start_metric(metric); end   # Sent before individual metric is measured.
  def finish_metric(metric); end   # Sent after individual metric measurement is complete.
  def finish; end           # Sent after metric_fu has completed all measurements.
  def display_results; end     # Used to open results in browser, etc.
end
```

Then

```shell
metric_fu --format MyCustomFormatter
```

See [lib/metric_fu/formatter](https://github.com/metricfu/metric_fu/tree/main/lib/metric_fu/formatter) for examples.

MetricFu will attempt to require a custom formatter by
fully qualified name based on ruby search path. So if you include a custom
formatter as a gem in your Gemfile, you should be able to use it out of the box.
But you may find in certain cases that you need to add a require to
your .metrics configuration file.

For instance, to require a formatter in your app's lib directory `require './lib/my_custom_formatter.rb'`

### Configure Graph Engine

By default, MetricFu uses the Bluff (JavaScript) graph engine.

```ruby
MetricFu.configuration.configure_graph_engine(:bluff)
```

But you can also use the [Highcharts JS library](https://shop.highsoft.com/)

```ruby
MetricFu.configuration.configure_graph_engine(:highcharts)
```

Notice: There was previously a `:gchart` option.
It was not properly deprecated in the 4.x series.

## Common problems / debugging

- ['ArgumentError; message invalid byte sequence in US-ASCII'](https://github.com/metricfu/metric_fu/issues/215) may be caused by having a default external encoding that is not UTF-8. You can see this in the output of `metric_fu --debug`
  - OSX: Ensure you have set `LANG=en_US.UTF-8` and `LC_ALL=en_US.UTF-8`. You can add these to your `~/.profile`.

## Compatibility

- It is currently testing on MRI (>= 1.9.3), JRuby (19 mode), and Rubinius (19 mode). Ruby 1.8 is no longer supported.

- For 1.8.7 support, see version 3.0.0 for partial support, or 2.1.3.7.18.1 (where [Semantic Versioning](http://semver.org/) goes to die)

- MetricFu no longer runs any of the analyzed code. For code coverage, you may use a formatter as documented above

- The Cane, Flog, and Rails Best Practices metrics are disabled when Ripper is not available

## Contributing

Take a look at our [contributing guide](https://github.com/metricfu/metric_fu/blob/master/CONTRIBUTING.md).
Bug reports and pull requests are welcome on GitHub at [https://github.com/metricfu/metric_fu](https://github.com/metricfu/metric_fu). This project is intended to be a safe, welcoming space for collaboration, and contributors are expected to adhere to the [Code of Conduct](https://www.contributor-covenant.org/version/1/4/code-of-conduct/).

## Resources

- [Wiki](https://github.com/metricfu/metric_fu/wiki)
- [List of code tools](https://github.com/metricfu/metric_fu/wiki/Code-Tools)
- [Roadmap](https://github.com/metricfu/metric_fu/wiki/Roadmap)
- [Google Group](https://groups.google.com/forum/#!forum/metric_fu)
- [Original repository by Jake Scruggs](https://github.com/jscruggs/metric_fu)
- [Jake's post about stepping down](http://jakescruggs.blogspot.com/2012/08/why-i-abandoned-metricfu.html)
