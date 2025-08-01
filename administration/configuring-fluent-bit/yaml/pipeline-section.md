# Pipeline

The `pipeline` section defines the flow of how data is collected, processed, and sent to its final destination. It encompasses the following core concepts:

| Name | Description |
| ---- | ----------- |
| `inputs` | Specifies the name of the plugin responsible for collecting or receiving data. This component serves as the data source in the pipeline. Examples of input plugins include `tail`, `http`, and `random`. |
| `processors` | **Unique to YAML configuration**, processors are specialized plugins that handle data processing directly attached to input plugins. Unlike filters, processors aren't dependent on tag or matching rules. Instead, they work closely with the input to modify or enrich the data before it reaches the filtering or output stages. Processors are defined within an input plugin section. |
| `filters` | Filters are used to transform, enrich, or discard events based on specific criteria. They allow matching tags using strings or regular expressions, providing a more flexible way to manipulate data. Filters run as part of the main event loop and can be applied across multiple inputs and filters. Examples of filters include `modify`, `grep`, and `nest`. |
| `outputs` | Defines the destination for processed data. Outputs specify where the data will be sent, such as to a remote server, a file, or another service. Each output plugin is configured with matching rules to determine which events are sent to that destination. Common output plugins include `stdout`, `elasticsearch`, and `kafka`. |

## Example configuration

{% hint style="info" %}

**Note:** Processors can be enabled only by using the YAML configuration format. Classic mode configuration format  doesn't support processors.

{% endhint %}

Here's an example of a pipeline configuration:

{% tabs %}
{% tab title="fluent-bit.yaml" %}

```yaml
pipeline:
    inputs:
        - name: tail
          path: /var/log/example.log
          parser: json

          processors:
              logs:
                  - name: record_modifier

    filters:
        - name: grep
          match: '*'
          regex: key pattern

    outputs:
        - name: stdout
          match: '*'
```

{% endtab %}
{% endtabs %}

## Pipeline processors

Processors operate on specific signals such as logs, metrics, and traces. They're attached to an input plugin and must specify the signal type they will process.

### Example of a processor

In the following example, the `content_modifier` processor inserts or updates (upserts) the key `my_new_key` with the value `123` for all log records generated by the tail plugin. This processor is only applied to log signals:

{% tabs %}
{% tab title="fluent-bit.yaml" %}

```yaml
parsers:
    - name: json
      format: json

pipeline:
      inputs:
          - name: tail
            path: /var/log/example.log
            parser: json

            processors:
                logs:
                    - name: content_modifier
                      action: upsert
                      key: my_new_key
                      value: 123

      filters:
          - name: grep
            match: '*'
            regex: key pattern

      outputs:
          - name: stdout
            match: '*'
```

{% endtab %}
{% endtabs %}

Here is a more complete example with multiple processors:

{% tabs %}
{% tab title="fluent-bit.yaml" %}

```yaml
service:
    log_level: info
    http_server: on
    http_listen: 0.0.0.0
    http_port: 2021

pipeline:
    inputs:
        - name: random
          tag: test-tag
          interval_sec: 1

          processors:
              logs:
                  - name: modify
                    add: hostname monox

                  - name: lua
                    call: append_tag
                    code: |
                      function append_tag(tag, timestamp, record)
                          new_record = record
                          new_record["tag"] = tag
                          return 1, timestamp, new_record
                      end

    outputs:
        - name: stdout
          match: '*'

          processors:
              logs:
                  - name: lua
                    call: add_field
                    code: |
                      function add_field(tag, timestamp, record)
                          new_record = record
                          new_record["output"] = "new data"
                          return 1, timestamp, new_record
                      end
```

{% endtab %}
{% endtabs %}

Processors can be attached to inputs and outputs.

### How processors are different from filters

While processors and filters are similar in that they can transform, enrich, or drop data from the pipeline, there is a significant difference in how they operate:

- Processors: Run in the same thread as the input plugin when the input plugin is configured to be threaded (threaded: true). This design provides better performance, especially in multi-threaded setups.

- Filters: Run in the main event loop. When multiple filters are used, they can introduce performance overhead, particularly under heavy workloads.

## Running filters as processors

You can configure existing [Filters](https://docs.fluentbit.io/manual/pipeline/filters) to run as processors. There are no specific changes needed; you use the filter name as if it were a native processor.

### Example of a filter running as a processor

In the following example, the `grep` filter is used as a processor to filter log events based on a pattern:

{% tabs %}
{% tab title="fluent-bit.yaml" %}

```yaml
parsers:
    - name: json
      format: json

pipeline:
    inputs:
        - name: tail
          path: /var/log/example.log
          parser: json

          processors:
              logs:
                  - name: grep
                    regex: log aa
    outputs:
        - name: stdout
          match: '*'
```

{% endtab %}
{% endtabs %}
