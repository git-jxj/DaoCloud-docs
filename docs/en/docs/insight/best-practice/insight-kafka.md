# Kafka + Elasticsearch Stream Architecture for Handling Large-Scale Logs

As businesses grow, the amount of log data generated by applications increases significantly.
To ensure that systems can properly collect and analyze massive amounts of log data,
it is common practice to introduce a streaming architecture using Kafka to handle asynchronous data collection.
The collected log data flows through Kafka and is consumed by corresponding components,
which then store the data into Elasticsearch for visualization and analysis using Insight.

This article will introduce two solutions:

- Fluentbit + Kafka + Logstash + Elasticsearch
- Fluentbit + Kafka + Vector + Elasticsearch

Once we integrate Kafka into the logging system, the data flow diagram looks as follows:

![logging-kafka](./images/logging-kafka.png)

Both solutions share similarities but differ in the component used to consume Kafka data.
To ensure compatibility with Insight's data analysis, the format of the data consumed from
Kafka and written into Elasticsearch should be consistent with the data directly written
by Fluentbit to Elasticsearch.

Let's first see how Fluentbit writes logs to Kafka:

## Modifying Fluentbit Output Configuration

Once the Kafka cluster is ready, we need to modify the content of the `insihgt-system` namespace's
`ConfigMap`. We will add three Kafka outputs and comment out the original three Elasticsearch outputs:

Assuming the Kafka Brokers address is: `insight-kafka.insight-system.svc.cluster.local:9092`

```console
    [OUTPUT]
        Name        kafka
        Match_Regex (?:kube|syslog)\.(.*)
        Brokers     insight-kafka.insight-system.svc.cluster.local:9092
        Topics      insight-logs
        format      json
        timestamp_key @timestamp
        rdkafka.batch.size 65536
        rdkafka.compression.level 6
        rdkafka.compression.type lz4
        rdkafka.linger.ms 0
        rdkafka.log.connection.close false
        rdkafka.message.max.bytes 2.097152e+06
        rdkafka.request.required.acks 1
    [OUTPUT]
        Name        kafka
        Match_Regex (?:skoala-gw)\.(.*)
        Brokers     insight-kafka.insight-system.svc.cluster.local:9092
        Topics      insight-gw-skoala
        format      json
        timestamp_key @timestamp
        rdkafka.batch.size 65536
        rdkafka.compression.level 6
        rdkafka.compression.type lz4
        rdkafka.linger.ms 0
        rdkafka.log.connection.close false
        rdkafka.message.max.bytes 2.097152e+06
        rdkafka.request.required.acks 1
    [OUTPUT]
        Name        kafka
        Match_Regex (?:kubeevent)\.(.*)
        Brokers     insight-kafka.insight-system.svc.cluster.local:9092
        Topics      insight-event
        format      json
        timestamp_key @timestamp
        rdkafka.batch.size 65536
        rdkafka.compression.level 6
        rdkafka.compression.type lz4
        rdkafka.linger.ms 0
        rdkafka.log.connection.close false
        rdkafka.message.max.bytes 2.097152e+06
        rdkafka.request.required.acks 1
```

Next, let's discuss the subtle differences in consuming Kafka data and writing it to Elasticsearch.
As mentioned at the beginning of this article, we will explore Logstash and Vector as two ways to consume Kafka data.

## Consuming Kafka and Writing to Elasticsearch

Assuming the Elasticsearch address is: `https://mcamel-common-es-cluster-es-http.mcamel-system:9200`

### Using Logstash for Consumption

If you are familiar with the Logstash technology stack, you can continue using this approach.

When deploying Logstash via Helm, you can add the following pipeline:

```yaml
# Allows you to add any config files in /usr/share/logstash/config/
# such as logstash.yml and log4j2.properties
#
# Note that when overriding logstash.yml, `http.host: 0.0.0.0` should always be included
# to make default probes work.
logstashConfig:
  logstash.yml: |
    http.host: 0.0.0.0
    xpack.monitoring.enabled: false
logstashPipeline:
  insight-logs.conf: |
    input {
      kafka {
        topics_pattern => "insight-logs"         # You can also use a wildcard for matching, like: all-log.*
        bootstrap_servers => "insight-kafka.insight-system.svc.cluster.local:9092"   # kafka IP Port
        enable_auto_commit => true
        #codec => json                               # data format
        consumer_threads => 1                       # The number of corresponding partitions
        decorate_events => true
        #auto_offset_rest => "latest"               # The default value
        #group_id => "all-logs-group"                # Kafka's consumption group
        codec => "plain"
      }
    }

    filter {
      mutate { gsub => [ "message", "@timestamp", "_@timestamp"] }
      json {source => "message"}
      date {
          match => [ "_@timestamp", "UNIX" ]
          remove_field => "_@timestamp"
          remove_tag => "_timestampparsefailure"
      }
      mutate {
          remove_field => ["event", "message"]
      }
    }
    output {
      elasticsearch {
        hosts => ["https://mcamel-common-es-cluster-es-http.mcamel-system:9200"]
        user => 'elastic'
        ssl => 'true'
        password => 'XAlJ948ZY0leE320SQ6hfv17'
        ssl_certificate_verification => 'false'
        index => "insight-es-k8s-logs-alias"
      }
    }

  insight-event.conf: |
    input {
      kafka {
        topics_pattern => "insight-event"         # You can also use a wildcard for matching, like: all-log.*
        bootstrap_servers => "insight-kafka.insight-system.svc.cluster.local:9092"   # kafka ip port
        enable_auto_commit => true
        #codec => json                               # data format
        consumer_threads => 1                       # The number of corresponding partitions
        decorate_events => true
        #auto_offset_rest => "latest"               # The default value
        #group_id => "all-logs-group"                # Kafka's consumption group
        codec => "plain"
      }
    }

    filter {
      mutate { gsub => [ "message", "@timestamp", "_@timestamp"] }
      json {source => "message"}
      date {
          match => [ "_@timestamp", "UNIX" ]
          remove_field => "_@timestamp"
          remove_tag => "_timestampparsefailure"
      }
      mutate {
          remove_field => ["event", "message"]
      }
    }
    output {
      elasticsearch {
        hosts => ["https://mcamel-common-es-cluster-es-http.mcamel-system:9200"]
        user => 'elastic'
        ssl => 'true'
        password => 'XAlJ948ZY0leE320SQ6hfv17'
        ssl_certificate_verification => 'false'
        index => "insight-es-k8s-event-logs-alias"
      }
    }

  insight-gw-skoala.conf: |
    input {
      kafka {
        topics_pattern => "insight-gw-skoala"         #  You can also use a wildcard for matching, like: all-log.*
        bootstrap_servers => "insight-kafka.insight-system.svc.cluster.local:9092"   # kafka ip port
        enable_auto_commit => true
        #codec => json                               # date format
        consumer_threads => 1                       # the number of corresponding partitions
        decorate_events => true
        #auto_offset_rest => "latest"               # The default value
        #group_id => "all-logs-group"                # Kafka's consumption group
        codec => "plain"
      }
    }

    filter {
      mutate { gsub => [ "message", "@timestamp", "_@timestamp"] }
      json {source => "message"}
      date {
          match => [ "_@timestamp", "UNIX" ]
          remove_field => "_@timestamp"
          remove_tag => "_timestampparsefailure"
      }
      mutate {
          remove_field => ["event", "message"]
      }
    }
    output {
      elasticsearch {
        hosts => ["https://mcamel-common-es-cluster-es-http.mcamel-system:9200"]
        user => 'elastic'
        ssl => 'true'
        password => 'XAlJ948ZY0leE320SQ6hfv17'
        ssl_certificate_verification => 'false'
        index => "skoala-gw-alias"
      }
    }
```

### Consumption with Vector

If you are familiar with the Vector technology stack, you can continue using this approach.

When deploying Vector via Helm, you can reference a ConfigMap configuration file with the following rules:

```yaml
metadata:
  name: vector
apiVersion: v1
data:
  aggregator.yaml: |
    api:
      enabled: true
      address: '0.0.0.0:8686'
    sources:
      insight_logs_kafka:
        type: kafka
        bootstrap_servers: 'insight-kafka.insight-system.svc.cluster.local:9092'
        group_id: consumer-group-insight
        topics:
          - insight-logs
      insight_event_kafka:
        type: kafka
        bootstrap_servers: 'insight-kafka.insight-system.svc.cluster.local:9092'
        group_id: consumer-group-insight
        topics:
          - insight-event
      insight_gw_skoala_kafka:
        type: kafka
        bootstrap_servers: 'insight-kafka.insight-system.svc.cluster.local:9092'
        group_id: consumer-group-insight
        topics:
          - insight-gw-skoala
    transforms:
      insight_logs_remap:
        type: remap
        inputs:
          - insight_logs_kafka
        source: |2
              . = parse_json!(string!(.message))
              .@timestamp = now()
      insight_event_kafka_remap:
        type: remap
        inputs:
          - insight_event_kafka
          - insight_gw_skoala_kafka
        source: |2
              . = parse_json!(string!(.message))
              .@timestamp = now()
      insight_gw_skoala_kafka_remap:
        type: remap
        inputs:
          - insight_gw_skoala_kafka
        source: |2
              . = parse_json!(string!(.message))
              .@timestamp = now()
    sinks:
      insight_es_logs:
        type: elasticsearch
        inputs:
          - insight_logs_remap
        api_version: auto
        auth:
          strategy: basic
          user: elastic
          password: 8QZJ656ax3TXZqQh205l3Ee0
        bulk:
          index: insight-es-k8s-logs-alias-1418
        endpoints:
          - 'https://mcamel-common-es-cluster-es-http.mcamel-system:9200'
        tls:
          verify_certificate: false
          verify_hostname: false
      insight_es_event:
        type: elasticsearch
        inputs:
          - insight_event_kafka_remap
        api_version: auto
        auth:
          strategy: basic
          user: elastic
          password: 8QZJ656ax3TXZqQh205l3Ee0
        bulk:
          index: insight-es-k8s-event-logs-alias-1418
        endpoints:
          - 'https://mcamel-common-es-cluster-es-http.mcamel-system:9200'
        tls:
          verify_certificate: false
          verify_hostname: false
      insight_es_gw_skoala:
        type: elasticsearch
        inputs:
          - insight_gw_skoala_kafka_remap
        api_version: auto
        auth:
          strategy: basic
          user: elastic
          password: 8QZJ656ax3TXZqQh205l3Ee0
        bulk:
          index: skoala-gw-alias-1418
        endpoints:
          - 'https://mcamel-common-es-cluster-es-http.mcamel-system:9200'
        tls:
          verify_certificate: false
          verify_hostname: false
```

## Checking if it's Working Properly

You can verify if the configuration is successful by checking if there are new data
in the Insight log query interface or observing an increase in the number of indices in Elasticsearch.

## References

- [Bitnami Logstash Helm Chart](https://github.com/bitnami/charts/tree/main/bitnami/logstash/#installing-the-chart)
- [Vector Helm Chart](https://vector.dev/docs/setup/installation/package-managers/helm/)
- [Vector Practice](https://wiki.eryajf.net/pages/0322lius/#_0-%E5%89%8D%E8%A8%80) (Chinese)
- [Vector Performance](https://github.com/vectordotdev/vector/blob/master/README.md)
