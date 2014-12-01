---
layout: bt_wiki
title: Diamond Plugin
category: Plugins
publish: true
abstract: "Cloudify diamond plugin description and configuration"
pageord: 200

---

{%summary%}Diamond plugin is used to install & configure a [Diamond](https://github.com/BrightcoveOS/Diamond) monitoring agent (version 3.5) on hosts{%endsummary%}

# Example
The following example shows the configuration possibilities of the plugin.

{% highlight yaml %}
node_types:
  my_type:
    derived_from: cloudify.nodes.WebServer
    properties:
      collectors_config: {}

node_templates:
  vm:
    type: cloudify.nodes.Compute
    interfaces:
      cloudify.interfaces.monitoring_agent:
        install:
          implementation: diamond.diamond_agent.tasks.install
          diamond_config:
            interval: 10
        start: diamond.diamond_agent.tasks.start
        stop: diamond.diamond_agent.tasks.stop
        uninstall: diamond.diamond_agent.tasks.uninstall

  app:
    type: my_type
    properties:
      collectors_config:
        CPUCollector: {}
        DiskUsageCollector:
          config:
            devices: x?vd[a-z]+[0-9]*$
        MemoryCollector: {}
        NetworkCollector: {}
        ExampleCollector:
          path: collectors/example.py
          config:
              key: value
    interfaces:
      cloudify.interfaces.monitoring:
        start:
          implementation: diamond.diamond_agent.tasks.add_collectors
          inputs:
            collectors_config: { get_propery: [SELF, collectors_config] }
        stop:
          implementation: diamond.diamond_agent.tasks.del_collectors
          inputs:
            collectors_config: { get_propery: [SELF, collectors_config] }
    relationships:
      - type: cloudify.relationships.contained_in
        target: node
{%endhighlight%}

# Interfaces
Two interfaces are involved in setting up a monitoring agent on a machine: `cloudify.interfaces.monitoring_agent` & `cloudify.interfaces.monitoring`.

`cloudify.interfaces.monitoring_agent` is the interface in charge of installing, starting stopping and uninstalling the agent.

`cloudify.interfaces.monitoring` is the interface in charge of configuring the monitoring agent.

The example above shows how the Diamond plugin maps to these interfaces.

# Global config
The Diamond agent has a number of configuration sections, some of them are global while other are relevant to specific components.
It is possible to pass [global config](https://github.com/BrightcoveOS/Diamond/blob/v3.5/conf/diamond.conf.example) setting via the `install` operation:
{% highlight yaml %}
interfaces:
  cloudify.interfaces.monitoring_agent:
    install:
      implementation: diamond.diamond_agent.tasks.install
      diamond_config:
        interval: 10
{%endhighlight%}
In the above example we set the [global poll interval](https://github.com/BrightcoveOS/Diamond/blob/v3.5/conf/diamond.conf.example#L176) to 10 seconds
(each collector will be polled for data every 10 seconds).

## Handler
The Handlers job in Diamond is to output collected data to different destinations. By default, the Diamond plugin will setup a custom handler which will output
the collected metrics to Cloudify's manager.

It is possible to set an alternative handler in case you want to output data into a different destination:
{% highlight yaml %}
interfaces:
  cloudify.interfaces.monitoring_agent:
    - install:
      mapping: diamond.diamond_agent.tasks.install
      properties:
        diamond_config:
          handlers:
            diamond.handler.graphite.GraphiteHandler:
              host: graphite.example.com
              port: 2003
              timeout: 15
{%endhighlight%}
In the example above we configured [handler for Graphite](https://github.com/BrightcoveOS/Diamond/wiki/handler-GraphiteHandler).

{%note title=Cloudify's default handler%}
If you wish to add your own handler but maintain Cloudify's default handler, see [this](https://github.com/cloudify-cosmo/cloudify-diamond-plugin/blob/1.1/diamond_agent/tasks.py#L38).
{%endnote%}

# Collectors config
Collectors are Diamond's data fetchers. Diamond comes with a large numbers of [built-in collectors](https://github.com/BrightcoveOS/Diamond/wiki/Collectors).

Collectors are added with the `install` operation of the `cloudify.interfaces.monitoring` interface:
{% highlight yaml %}
interfaces:
  cloudify.interfaces.monitoring:
    start:
      implementation: diamond.diamond_agent.tasks.add_collectors
      inputs:
        collectors_config:
          CPUCollector: {}
          DiskUsageCollector:
            config:
              devices: x?vd[a-z]+[0-9]*$
          MemoryCollector: {}
          NetworkCollector: {}
{%endhighlight%}
In the example above we configure 4 collectors: [CPUCollector](https://github.com/BrightcoveOS/Diamond/wiki/collectors-CPUCollector),
[DiskUsageCollector](https://github.com/BrightcoveOS/Diamond/wiki/collectors-DiskUsageCollector),
[MemoryCollector](https://github.com/BrightcoveOS/Diamond/wiki/collectors-MemoryCollector) and
[NetworkCollector](https://github.com/BrightcoveOS/Diamond/wiki/collectors-NetworkCollector).

It is also possible to add collector specific configuration via the `config` dictionary (as with `DiskUsageCollector`). If `config` is not provided, the collector will use its default settings.

{%note title=Default config values%}
Config values are left with their default values unless explicitly overwritten.
{%endnote%}

# Custom collectors & handlers
Collectors & handlers are essentially Python modules that implements specific Diamond interfaces.

It is possible to create your own collectors or handlers and configure them in Diamond. The example below shows how to upload a custom collector:
{% highlight yaml %}
collectors_config:
  ExampleCollector:
    path: collectors/example.py
      config:
        key: value
{%endhighlight%}

`path` points to the location of your custom collector (relative location to blueprint directory root). `ExampleCollector` is the name of the main class inside `example.py` that extends `diamond.collector.Collector`.

Providing a custom handler is done in a similar manner:
{% highlight yaml %}
diamond_config:
  handlers:
    example_handler.ExampleHandler:
      path: handlers/example_handler.py
      config:
        key: value
{%endhighlight%}

where `example_handler` is the name of the file and `ExampleHandler` is the name of the class that extends `diamond.handler.Handler`.

Note that handlers are configured as part of the `global config`.

{%note title=Collectors & Handlers prerequisite%}
Diamond's wide range of collectors, handlers and extensibility possibilities comes with a price - It's not always promised that you'll have all the required dependencies built in on your instance.

For example, you might find yourself trying to use `MongoDBCollector` collector which imports [pymongo](http://api.mongodb.org/python/current/) module internally.
Since `pymongo` is not part of the Python standard library, this will fail unless you will install it separately. See nodecellar example.
{%endnote%}