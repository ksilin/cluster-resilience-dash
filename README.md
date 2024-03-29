## Resilience dashboard

This project provides examples of dashboards for improving the resileince and usage patterns of shared resources, such as Confluent Cloud clusters. 

Please note that this is not an exhaustive list and further metrics can be added for better observability of shared resource usage.   


### Data Sources

Two data sources are used: 

#### Confluent Cloud metrics API

Easily accessible to a central team and can be used to monitor client activity across enviroments, clusters and domains. 

Can be accesses via scraping for lower resolution metrics as well as queries for higher, adjustable resolution and higher cardinality.

Not all important metrics are accessible. 

A template for a Prometheus scraper job for the Confluent Cloud Metrics API: 

```yaml
  - job_name: Confluent Cloud
    scrape_interval: 1m
    scrape_timeout: 1m
    honor_timestamps: true
    static_configs:
      - targets:
        - api.telemetry.confluent.cloud
    scheme: https
    basic_auth:
      username: CCLOUD_API_KEY
      password: CCLOUD_SECRET
    metrics_path: /v2/metrics/cloud/export
    params:
      resource.kafka.id: [lkc-xxx]  
```

#### Kafka Client JMX metrics

Need to be explicitly exposed by the clients and made available on pre-arranged endpoints. 

Can provide a high-resolution and detailed insight into the inner workings and connectivity of clients.

A template for a Prometheus scraper job for Client JMX metrics provided by a Quarkus-based application: 

```yaml
  - job_name: "quarkus_app"
    scrape_interval: 1m
    scrape_timeout: 30s
    metrics_path: /q/metrics/prometheus
    scheme: http
    static_configs:
      - targets: 
        - localhost:8080
```

### Prerequisites for improved results

Most metrics are grouped by a number of attributes. Confluent Cloud metrics are often grouped by the cluster ID and the principal - typically a Service Account ID. These are Confluent-specific identifiers. 

Thus, additional information is required to directly act on these metrics. In this case:
* a mapping from a cluster ID to a cluster name and environment. 
* a mapping from the environment and cluster ID to the responsible and their contact data.  
* a mapping from the service account ID to an application.
* a mapping from the application to the responsbile team and their contact data. 

Additionally, a fine-granular mapping of service accounts to individual applications can help in pinpointing and addressing the issue quicker. 

For most of the follwing metrics, it is not possible to define a unified value to alert on, due to different application patterns and expecteations. The limits should be defined iteratively, in collaboration between the client and the cluster/observability/governance teams. 

## Confluent Cloud metrics

### -- LeaveGroup & JoinGroup request rate

#### What? 

These requests are emitted by clients on consumer group membership changes. This happens both on voluntary and involuntary changes. Examples for voluntary changes are starting, stopping or resizing an application. Involuntary changes are usually failures - in the application itself or in the connectivity between the application and the cluster.  


#### Why

Unexpected LeaveGroup&JoinGroup requests (especially JoinGroup, since a failing client may not always send a LeaveGroup request) can indicate issues with applications. When processing times out, poison pills are processed, clients will be excluded from consumer groups, triggering rebalances. 

Clients with these issues tend to create an unnecessary load on the cluster, reprocessing records multiple times, without making progress. 

Typical issues with the clients leading to these issues include communicating with external systems in consumer loops in an unbounded way, processing too many records in a single processing loop, not handling deserialization and preocessing exceptions.   

#### How? 

`sum by(principal_id, kafka_id) (rate(confluent_kafka_server_request_count{type=~"JoinGroup|LeaveGroup"}[10m]))`

It is difficult to define a specific critical value for alerting for this metric, since larger-scale, or repeated events, such as starting or restarting of applications can lead to false-positive alerts.

To make this metric actionable, correlation with other deployment parameters need to be made, e.g. when were applications deployed, redeployed, restarted. 

It is difficult to be sure, whether a number of `JoinGroup`requests can be traced back to a genuine issue. Some hints may be found below. 

* Client error metrics increase before the JoinGroup request can indicate a genuine problem. 
* The burst itself is short and not repeated. Errors often result in periodic rebalances, or continuous, long rebalances. 
* can these JoinGroup requests be correlated with a consumer group member being added to the group without existing IDs being removed? This may be explained by scaling up a consumer group, or deploying a new applicaiton or a new version of an existing application.   
* are deployment schedules known, or ad-hoc deployments communicated? JoinGroup request bursts correlating with these events are probably benign. 
* analyzing the logs of the clients can help understand what happened.

### -- Metadata requests per minute

![alt text](resources/metadata.requests.png)

#### What

To send produce and consume rquests to correct brokers, clients need to know which brokers own which partitions. This information is provided via metadata responses to metadata requests.

#### Why

A client exhibiting a high rate of metadata requests might have troubles connecting to the right broker. Perhaps the topics it needs to work properly do not exist. These clients can generate load on the cluster via repeated requests without performing any work.   

Unexecpectedly high rate of metadata requests should be investigated to ensure correct configuration and provisioning of required resources.   

#### How

`sum by(type, principal_id, kafka_id, ) (confluent_kafka_server_request_count{type="Metadata"})`

No specific clitical value for alerting can be reaonably defined across clients. A relatively high rate of metadata requests should be verifies with the clients implementing team. Some clients may implement a liveness probe that works in terms of metadata requests.


### -- Offset commit rate

#### What

Consumers need to commit the offsets to record the current progress of record processing. Should the consumer fail, it will receive records starting from the last committed offset after restarting.   

The broker-side consumer lag will decrease only after the consumer committed offsets. 

#### Why 

Each offset commit generates additional cluster load. 

One reason for an high and unwarrented rate of offset commits is commiitting offsets after every processed record. While this approach can reduce teh cumber of reprocessed records in case of an error (the usual number of reprocessed records woulb be `max.poll.records`), this also constantly generates a load on the cluster.   

#### How

`sum by(principal_id, kafka_id) (rate(confluent_kafka_server_request_count{type="OffsetCommit"}[10m]))`

There is no one-size-fits-all rate to fit all clients. We recommend comparing rates between clients and over time and clarify client-specific requirements.


### -- Heartbeats per minute

#### What

Heartbeats are sent by consumers as one of the two liveness criteria, in addition to the actual polling for records. While polling for records tends to have rather high timeouts on order of minutes, hearbeat is used to detect failed consumers quickly, typically after 10 seconds or 3 missed heartbeats. 

#### Why 

When client configuration is modified, we sometimes see heartbeat intervals being reduced, perhaps for even faster failure detection. This is generally not recommended, as it puts an additional load on the cluster. While small for any individual client, this load can add up to significant one in a shared cluster.  

#### How

`sum by(type, principal_id, kafka_id) (rate(confluent_kafka_server_request_count{type="Heartbeat"}[1m]))`


### -- Bytes per request estimate

#### What

This is a synthetic metric, created as a rate of incoming bytes over the rate of produce requests.

#### Why 

Applications producing a high number of small requests might increase the cluster load. On frequent small requests, it is recommended to review the client batch settigns.

#### How

`sum(rate(confluent_kafka_server_received_bytes[5m])) / on(kafka_id) sum(rate(confluent_kafka_server_request_count{type="Produce"}[5m]))`

Please note the use of `confluent_kafka_server_received_bytes` and the limitation to Produce requests here. This is the actual data produced, contrary to the more general `confluent_kafka_server_request_bytes`, which takes in the account all requests. 


### -- Average record size estimate

![alt text](resources/avg.record.size.png)

#### What

This is a synthetic metric, created as a rate of bytes over the rate of records produced.

#### Why 

Identifying applications producing very large records may help in reducing cluster load. Very large records are rarely used in their entirely for processing. For large amount of binary data, patterns such as claim check are recommended instead.    

#### How

`sum by(kafka_id, topic) (rate(confluent_kafka_server_received_bytes[5m])) / sum by(kafka_id, topic) (rate(confluent_kafka_server_received_records[5m]))`

Please note the use of `confluent_kafka_server_received_bytes` and the limitation to Produce requests here. This is the actual data produced, contrary to the more general `confluent_kafka_server_request_bytes`, which takes in the account all requests. 


### -- Records per produce request estimate

#### What

This is a synthetic metric, created as a rate of records over the rate of produce requests.

#### Why 

Applications producing a low number of records per produce request might exhibit suboptimal batching and compression, additionally increasing the cluster load through a high number of small produce requests.  

#### How

`sum by(kafka_id) (rate(confluent_kafka_server_received_records[5m])) / on(kafka_id) sum by(kafka_id) (rate(confluent_kafka_server_request_count{type="Produce"}[5m]))`

## Client metrics


### -- Client failed authentication rate

#### What

Aggregates consumer, producer and admin client metrics.

#### Why 

A client failing authentication should be a rare, intermittent occurrence. A client consistently failing authentication generates unnecessary load on the cluster without performing any useful work. 

Often, failed authentication is due to issues with credential provisioning, e.g. keys being rotated. 

#### How

`kafka_consumer_failed_authentication_rate`, `kafka_producer_failed_authentication_rate`, `kafka_admin_client_failed_authentication_rate`

Short spikes can be disregarded, but longer (> 10 seconds), sustained failures should lead to a client being halted and the error investigated and addressed.


### -- Producer record errors

#### What

Records which were not written to the target topic. 

#### Why 

Record errors can lead to data loss, and data loss can be a significant source of business errors.  

#### How

Any number above zero should be investigated, except in cases where data loss is acceptable (e.g. low-value sensors).


### -- Consumer rebalances per hour

#### What

This is the client-side variant of LeaveGroup/JoinGroup request monitoring, directly measuring the rebalances, responsible for triggering the LeaveGroup/JoinGroup requests.  

#### Why 

Consumer groups are expected to rebalance rarely, when the consumer group resizes. Consumer groups rebalancing frequently or constantly can mean a problem in the processing or the data. Rebalancing consumers will reprocess data without making significant progress, potentially even generating a constent stream of duplicates downstream. 

#### How

`kafka_consumer_coordinator_rebalance_rate_per_hour`

Please note that the cooperative rebalance protocol for Kafka Streams may trigger probing rebalances while waiting for a task on the target instance to catch up, in order to prevent processing interruptions. Probing rebalances happen every 10 mintes by default, until a perfect balance has been achieved.  


### -- Producer compression rate

#### What

A direct measure of If an error in a Kafka Steams thread was not handled, it will cause the thread to fail. 
Without compression, the rate is constantly at 1.0.

#### Why 

Using producer compression is generally recommended, as it can lead to a more efficient use of resources on the cluster as well as in downstream clients. 

Tracking compression rate can help discover clients not using compression, as well as, in combination with other metrics, clients using compression suboptimally, e.g. with small batches.  

#### How

`kafka_producer_compression_rate_avg`

Compression efficiency is higly dependent on the algorithm used. Different algorithms, such as gzip, lz4 etc, make different trade-offs in terms of CPU usage and compression effectiveness. 

Even more than the algorithm, the data influences the effectiveness of compression. Textual data compresses very well, while binary data may not.

Compression happens on batch level, so larger batches 

Please note, that despite higher compression rates with textual messages, binary encoding with compression are generally still significantly more space-efficient.  


### -- Throttle time ms

#### What

When clients reach their alotted throuhgput quotas, they are required to wait before making the next request. 

#### Why

Clients being throttled demonstrates a non-alignment between client and cluster management. Throttled clients may genuinely need to be able to transfer more data, or may be required to consult the central/managing team for alignment.

#### How

`kafka_consumer_fetch_manager_fetch_throttle_time_avg`, `kafka_consumer_fetch_manager_fetch_throttle_time_max`, `kafka_producer_produce_throttle_time_avg`, `kafka_producer_produce_throttle_time_max`

A central management of client quotas with low defaults is recommended for shared clusters. The more broad the sharing, the more restricitve the quota handling should be in terms of both limits and process. 

Clients needing higher quotas are required to pass through a coordination process, creating an alignment of expectations and recoding the design decisions along with the quotas.   

### -- Failed stream threads

#### What

Only relevant for Kafka Streams client. 

If an error in a Kafka Steams thread was not handled, it will cause the thread to fail. 

#### Why 

Failed stream threads can lead to reblances, reprocessing, reduced rate of processing and should be investigated. 


### -- Task punctuate latency

#### What

Only relevant for Kafka Streams client. 

Punctuators are periodic tasks executed by Kafka Streams, mostly to perform tasks on data in state stores, e.g. deduplication, publishing aggregations, detencting missing events etc.

This metric records the duration of these tasks.  

#### Why 

Punctuation (periodic tasks) happens on the main thread in Kafka Streams. If it takes too long, the task may time out, leading to a rebalance. This is especially relevant for EOS with Kafka Streams. 

#### How

Please restrict the execution time of each punctuation, resuming on the next iteration. If this leads to furhter complications (e.g.the state store being analyzed too rarely), this issue may warrant a redesign of the application, or at least the partitioning.     


## Alerting & Actions

The dashboards do not provide any alerting capabilities. However, exemplary visual thresholds are defined for most metrics


### Future work

* Introduce alerting integration

* Add further client metrics

* Add further screenshots


