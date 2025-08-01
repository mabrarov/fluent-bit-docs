# Buffering and storage

<img referrerpolicy="no-referrer-when-downgrade" src="https://static.scarf.sh/a.png?x-pxid=cde12327-09ed-409c-ac02-7c0afa5eff51" />

[Fluent Bit](https://fluentbit.io) collects, parses, filters, and ships logs to a central place. A critical piece of this workflow is the ability to do _buffering_: a mechanism to place processed data into a temporary location until is ready to be shipped.

By default when Fluent Bit processes data, it uses Memory as a primary and temporary place to store the records. There are scenarios where it would be ideal to have a persistent buffering mechanism based in the filesystem to provide aggregation and data safety capabilities.

Choosing the right configuration is critical and the behavior of the service can be conditioned based in the backpressure settings. Before jumping into the configuration it helps to understand the relationship between _chunks_, _memory_, _filesystem_, and _backpressure_.

## Chunks, memory, filesystem, and backpressure

Understanding chunks, buffering, and backpressure is critical for a proper configuration.

### Backpressure

See [Backpressure](https://docs.fluentbit.io/manual/administration/backpressure) for a full explanation.

### Chunks

When an input plugin source emits records, the engine groups the records together in a _chunk_. A chunk's size usually is around 2&nbsp;MB. By configuration, the engine decides where to place this chunk. By default, all chunks are created only in memory.

### Irrecoverable chunks

There are two scenarios where Fluent Bit marks chunks as irrecoverable:

- When Fluent Bit encounters a bad layout in a chunk. A bad layout is a chunk that doesn't conform to the expected format. [Chunk definition](https://github.com/fluent/fluent-bit/blob/master/CHUNKS.md)

- When Fluent Bit encounters an incorrect or invalid chunk header size.

In both scenarios Fluent Bit logs an error message and then discards the irrecoverable chunks.

#### Buffering and memory

As mentioned previously, chunks generated by the engine are placed in memory by default, but this is configurable.

If memory is the only mechanism set for the input plugin, it will store as much data as possible in memory. This is the fastest mechanism with the least system overhead. However, if the service isn't able to deliver the records fast enough, Fluent Bit memory usage increases as it accumulates more data than it can deliver.

In a high load environment with backpressure, having high memory usage risks getting killed by the kernel's OOM Killer. To work around this backpressure scenario, limit the amount of memory in records that an input plugin can register using the `mem_buf_limit` property. If a plugin has queued more than the `mem_buf_limit`, it won't be able to ingest more until that data can be delivered or flushed properly. In this scenario the input plugin in question is paused. When the input is paused, records won't be ingested until the plugin resumes. For some inputs, such as TCP and tail, pausing the input will almost certainly lead to log loss. For the tail input, Fluent Bit can save its current offset in the current file it's reading, and pick back up when the input resumes.

Look for messages in the Fluent Bit log output like:

```text
[input] tail.1 paused (mem buf overlimit)
[input] tail.1 resume (mem buf overlimit)
```

Using `mem_buf_limit` is good for certain scenarios and environments. It helps to control the memory usage of the service. However, if a file rotates while the plugin is paused, data can be lost since it won't be able to register new records. This can happen with any input source plugin. The goal of `mem_buf_limit` is memory control and survival of the service.

For a full data safety guarantee, use filesystem buffering.

Choose your preferred format for an example input definition:

{% tabs %}
{% tab title="fluent-bit.yaml" %}

```yaml
pipeline:
  inputs:
    - name: tcp
      listen: 0.0.0.0
      port: 5170
      format: none
      tag: tcp-logs
      mem_buf_limit: 50MB
```

{% endtab %}
{% tab title="fluent-bit.conf" %}

```text
[INPUT]
  Name          tcp
  Listen        0.0.0.0
  Port          5170
  Format        none
  Tag           tcp-logs
  Mem_Buf_Limit 50MB
```

{% endtab %}
{% endtabs %}

If this input uses more than 50&nbsp;MB memory to buffer logs, you will get a warning like this in the Fluent Bit logs:

```text
[input] tcp.1 paused (mem buf overlimit)
```

{% hint style="info" %}

`m em_buf_Limit` applies only when `storage.type` is set to the default value of `memory`.

{% endhint %}

#### Filesystem buffering

Filesystem buffering helps with backpressure and overall memory control. Enable it using `storage.type filesystem`.

Memory and filesystem buffering mechanisms aren't mutually exclusive. Enabling filesystem buffering for your input plugin source can improve both performance and data safety.

Enabling filesystem buffering changes the behavior of the engine. Upon chunk creation, the engine stores the content in memory and also maps a copy on disk through [mmap(2)](https://man7.org/linux/man-pages/man2/mmap.2.html). The newly created chunk is active in memory, backed up on disk, and called to be `up`, which means the chunk content is up in memory.

Fluent Bit controls the number of chunks that are `up` in memory by using the filesystem buffering mechanism to deal with high memory usage and backpressure.

By default, the engine allows a total of 128 chunks `up` in memory in total, considering all chunks. This value is controlled by the service property `storage.max_chunks_up`. The active chunks that are `up` are ready for delivery and are still receiving records. Any other remaining chunk is in a `down` state, which means that it's only in the filesystem and won't be `up` in memory unless it's ready to be delivered. Chunks are never much larger than 2&nbsp;MB, so with the default `storage.max_chunks_up` value of 128, each input is limited to roughly 256&nbsp;MB of memory.

If the input plugin has enabled `storage.type` as `filesystem`, when reaching the `storage.max_chunks_up` threshold, instead of the plugin being paused, all new data will go to chunks that are `down` in the filesystem. This lets you control memory usage by the service and also provides a guarantee that the service won't lose any data. By default, the enforcement of the `storage.max_chunks_up` limit is best-effort. Fluent Bit can only append new data to chunks that are `up`. When the limit is reached chunks will be temporarily brought `up` in memory to ingest new data, and then put to a `down` state afterwards. In general, Fluent Bit works to keep the total number of `up` chunks at or under `storage.max_chunks_up`.

If `storage.pause_on_chunks_overlimit` is enabled (default is off), the input plugin pauses upon exceeding `storage.max_chunks_up`. With this option, `storage.max_chunks_up` becomes a hard limit for the input. When the input is paused, records won't be ingested until the plugin resumes. For some inputs, such as TCP and tail, pausing the input will almost certainly lead to log loss. For the tail input, Fluent Bit can save its current offset in the current file it's reading, and pick back up when the input is resumed.

Look for messages in the Fluent Bit log output like:

```text
[input] tail.1 paused (storage buf overlimit
[input] tail.1 resume (storage buf overlimit
```

##### Limiting filesystem space for chunks

Fluent Bit implements the concept of logical queues. Based on its tag, a chunk can be routed to multiple destinations. Fluent Bit keeps an internal reference from where a chunk was created and where it needs to go.

It's common to find cases where multiple destinations with different response times exist for a chunk, or one of the destinations is generating backpressure.

To limit the amount of filesystem chunks logically queueing, Fluent Bit v1.6 and later includes the `storage.total_limit_size` configuration property for output. This property limits the total size in bytes of chunks that can exist in the filesystem for a certain logical output destination. If one of the destinations reaches the configured `storage.total_limit_size`, the oldest chunk from its queue for that logical output destination will be discarded to make room for new data.

## Configuration

The storage layer configuration takes place in three sections:

- Service
- Input
- Output

The known Service section configures a global environment for the storage layer, the Input sections define which buffering mechanism to use, and the Output defines limits for the logical filesystem queues.

### Service section configuration

The Service section refers to the section defined in the main [configuration file](configuring-fluent-bit/classic-mode/configuration-file.md):

| Key | Description | Default |
| :--- | :--- | :--- |
| `storage.path` | Set an optional location in the file system to store streams and chunks of data. If this parameter isn't set, Input plugins can only use in-memory buffering. | _none_ |
| `storage.sync` | Configure the synchronization mode used to store the data in the file system. Using `full` increases the reliability of the filesystem buffer and ensures that data is guaranteed to be synced to the filesystem even if Fluent Bit crashes. On Linux, `full` corresponds with the `MAP_SYNC` option for [memory mapped files](https://man7.org/linux/man-pages/man2/mmap.2.html). Accepted values: `normal`, `full`. | `normal` |
| `storage.checksum` | Enable the data integrity check when writing and reading data from the filesystem. The storage layer uses the CRC32 algorithm. Accepted values: `Off`, `On`. | `Off` |
| `storage.max_chunks_up` | If the input plugin has enabled `filesystem` storage type, this property sets the maximum number of chunks that can be `up` in memory. Use this setting to control memory usage when you enable `storage.type filesystem`. | `128` |
| `storage.backlog.mem_limit` | If `storage.path` is set, Fluent Bit looks for data chunks that weren't delivered and are still in the storage layer. These are called _backlog_ data. _Backlog chunks_ are filesystem chunks that were left over from a previous Fluent Bit run; chunks that couldn't be sent before exit that Fluent Bit will pick up when restarted. Fluent Bit will check the `storage.backlog.mem_limit` value against the current memory usage from all `up` chunks for the input. If the `up` chunks currently consume less memory than the limit, it will bring the _backlog_ chunks up into memory so they can be sent by outputs. | `5M` |
| `storage.backlog.flush_on_shutdown` | When enabled, Fluent Bit will attempt to flush all backlog filesystem chunks to their destination during the shutdown process. This can help ensure data delivery before Fluent Bit stops, but can increase shutdown time. Accepted values: `Off`, `On`. | `Off` |
| `storage.metrics` | If `http_server` option is enabled in the main `[SERVICE]` section, this option registers a new endpoint where internal metrics of the storage layer can be consumed. For more details refer to the [Monitoring](monitoring.md) section. | `off` |
| `storage.delete_irrecoverable_chunks` | When enabled, [irrecoverable chunks](./buffering-and-storage.md#irrecoverable-chunks) will be deleted during runtime, and any other irrecoverable chunk located in the configured storage path directory will be deleted when Fluent Bit starts. Accepted values: `Off`, 'On`. | `Off` |

A Service section will look like this:

{% tabs %}
{% tab title="fluent-bit.yaml" %}

```yaml
service:
  flush: 1
  log_level: info
  storage.path: /var/log/flb-storage/
  storage.sync: normal
  storage.checksum: off
  storage.backlog.mem_limit: 5M
  storage.backlog.flush_on_shutdown: off
```

{% endtab %}
{% tab title="fluent-bit.conf" %}

```text
[SERVICE]
  flush                     1
  log_Level                 info
  storage.path              /var/log/flb-storage/
  storage.sync              normal
  storage.checksum          off
  storage.backlog.mem_limit 5M
  storage.backlog.flush_on_shutdown off
```

{% endtab %}
{% endtabs %}

This configuration sets an optional buffering mechanism where the route to the data is `/var/log/flb-storage/`. It uses `normal` synchronization mode, without running a checksum and up to a maximum of 5&nbsp;MB of memory when processing backlog data.

### Input section configuration

Optionally, any Input plugin can configure their storage preference. The following table describes the options available:

| Key | Description | Default |
| :--- | :--- | :--- |
| `storage.type` | Specifies the buffering mechanism to use. Accepted values: `memory`, `filesystem`. | `memory` |
| `storage.pause_on_chunks_overlimit` | Specifies if the input plugin should pause (stop ingesting new data) when the `storage.max_chunks_up` value is reached. |`off` |

The following example configures a service offering filesystem buffering capabilities and two input plugins being the first based in filesystem and the second with memory only.

{% tabs %}
{% tab title="fluent-bit.yaml" %}

```yaml
service:
  flush: 1
  log_level: info
  storage.path: /var/log/flb-storage/
  storage.sync: normal
  storage.checksum: off
  storage.max_chunks_up: 128
  storage.backlog.mem_limit: 5M

pipeline:
  inputs:
    - name: cpu
      storage.type: filesystem

    - name: mem
      storage.type: memory
```

{% endtab %}
{% tab title="fluent-bit.conf" %}

```text
[SERVICE]
  flush                     1
  log_Level                 info
  storage.path              /var/log/flb-storage/
  storage.sync              normal
  storage.checksum          off
  storage.max_chunks_up     128
  storage.backlog.mem_limit 5M

[INPUT]
  name          cpu
  storage.type  filesystem

[INPUT]
  name          mem
  storage.type  memory
```

{% endtab %}
{% endtabs %}

### Output section configuration

If certain chunks are filesystem `storage.type` based, it's possible to control the size of the logical queue for an output plugin. The following table describes the options available:

| Key | Description | Default |
| :--- | :--- | :--- |
| `storage.total_limit_size` | Limit the maximum disk space size in bytes for buffering chunks in the filesystem for the current output logical destination. | _none_ |

The following example creates records with CPU usage samples in the filesystem which are delivered to Google Stackdriver service while limiting the logical queue (buffering) to `5M`:

{% tabs %}
{% tab title="fluent-bit.yaml" %}

```yaml
service:
  flush: 1
  log_level: info
  storage.path: /var/log/flb-storage/
  storage.sync: normal
  storage.checksum: off
  storage.max_chunks_up: 128
  storage.backlog.mem_limit: 5M

pipeline:
  inputs:
    - name: cpu
      storage.type: filesystem

  outputs:
    - name: stackdriver
      match: '*'
      storage.total_limit_size: 5M
```

{% endtab %}
{% tab title="fluent-bit.conf" %}

```text
[SERVICE]
  flush                     1
  log_Level                 info
  storage.path              /var/log/flb-storage/
  storage.sync              normal
  storage.checksum          off
  storage.max_chunks_up     128
  storage.backlog.mem_limit 5M

[INPUT]
  name                      cpu
  storage.type              filesystem

[OUTPUT]
  name                      stackdriver
  match                     *
  storage.total_limit_size  5M
```

{% endtab %}
{% endtabs %}

If Fluent Bit is offline because of a network issue, it will continue buffering CPU
samples, keeping a maximum of 5&nbsp;MB of the newest data.
