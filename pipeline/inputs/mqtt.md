# MQTT

The _MQTT_ input plugin retrieves messages and data from MQTT control packets over a TCP connection. The incoming data to receive must be a JSON map.

## Configuration parameters

The plugin supports the following configuration parameters:

| Key           | Description                                                                                             | Default   |
|:--------------|:--------------------------------------------------------------------------------------------------------|:----------|
| `Listen`      | Listener network interface.                                                                             | `0.0.0.0` |
| `Port`        | TCP port where listening for connections.                                                               | `1883`    |
| `Payload_Key` | Specify the key where the payload key/value will be preserved.                                          | _none_    |
| `Threaded`    | Indicates whether to run this input in its own [thread](../../administration/multithreading.md#inputs). | `false`   |

## Get started

To listen for MQTT messages, you can run the plugin from the command line or through the configuration file.

### Command line

The MQTT input plugin lets Fluent Bit behave as a server. Dispatch some messages using a MQTT client. In the following example, the `mosquitto` tool is being used for the purpose:

Running the following command:

```shell
fluent-bit -i mqtt -t data -o stdout -m '*'
```

Returns a response like the following:

```text
...
[0] data: [1463775773, {"topic"=>"some/topic", "key1"=>123, "key2"=>456}]
...
```

The following command line will send a message to the MQTT input plugin:

```shell
mosquitto_pub  -m '{"key1": 123, "key2": 456}' -t some/topic
```

### Configuration file

In your main configuration file append the following:

{% tabs %}
{% tab title="fluent-bit.yaml" %}

```yaml
pipeline:
  inputs:
    - name: mqtt
      tag: data
      listen: 0.0.0.0
      port: 1883

  outputs:
    - name: stdout
      match: '*'
```

{% endtab %}
{% tab title="fluent-bit.conf" %}

```text
[INPUT]
  Name   mqtt
  Tag    data
  Listen 0.0.0.0
  Port   1883

[OUTPUT]
  Name   stdout
  Match  *
```

{% endtab %}
{% endtabs %}
