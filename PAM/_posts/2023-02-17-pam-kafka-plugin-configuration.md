---
layout: post_with_sidebar
title: How to setup PAM Kafka Extension with custom configuration parameters
category: PAM
subcategory: Integration

tags: [PAM, Integration, Kafka ]
---

Following the official documentation, only a few parameters could be specified for configuring integration with AMQ Streams.

after enabling the integration by setting **org.kie.kafka.server.ext.disabled** to **false** the following parameters could be specified:

| PARAMETER | DESCRIPTION | DEFAULT VALUE | OPTIONAL |
| --------- | ----------- | ------------- | -------- |
| org.kie.server.jbpm-kafka.ext.bootstrap.servers | comma-separated list of host and port of the broker. | localhost:9092 | false |
| org.kie.server.jbpm-kafka.ext.client.id | An identifier string to pass to the broker when making requests | | true |
| org.kie.server.jbpm-kafka.ext.topics.* | Mapping of message names to topic names | If a message name is not mapped using a system property, the process engine uses this name as the topic name. | true |
| org.kie.server.jbpm-kafka.ext.allow.auto.create.topics | Allow automatic topic creation | true | true |
| org.kie.server.jbpm-kafka.ext.group.id | A unique string that identifies the group to which this Kafka message consumer belong | jbpm-consumer | true |
| org.kie.server.jbpm-kafka.ext.acks | The number of acknowledgements that the Kafka leader must receive before marking the request as complete | 1 | true |
| org.kie.server.jbpm-kafka.ext.max.block.ms | The number of milliseconds for which the publish method blocks in milliseconds | 2000 | true

indeed this was true only before RHPAM 7.12, because in JBPM 7.56 the class handling the properties changed allowing the user to specify every Kafka client attribute by prefixing it with **org.kie.server.jbpm-kafka.ext.**

for exaple in order to configure the connectivity using SSL and SCRAM autentication the system properties will be:
~~~
org.kie.server.jbpm-kafka.ext.client.id="rhpam-test"
org.kie.server.jbpm-kafka.ext.bootstrap.servers="my-kafka-cluster-bootstrap:9093"
org.kie.server.jbpm-kafka.ext.acks="all"
org.kie.server.jbpm-kafka.ext.sasl.mechanism="SCRAM-SHA-512"
org.kie.server.jbpm-kafka.ext.security.protocol="SASL_SSL"
org.kie.server.jbpm-kafka.ext.sasl.jaas.config='org.apache.kafka.common.security.scram.ScramLoginModule required username="test-user" password="********";'
org.kie.server.jbpm-kafka.ext.ssl.truststore.location="/trust/store/mount/path/server-ca.p12"
org.kie.server.jbpm-kafka.ext.ssl.truststore.password="********"
org.kie.server.jbpm-kafka.ext.ssl.truststore.type="PKCS12"
~~~
further parameters can be found in [Official Kafka Documentation (producer configs)][kafka-doc-producer] and [Official Kafka Documentation (consumer configs)][kafka-doc-consumer]

even if both consumer and producer properties are specified using the same configuration pattern, the extension will configure the producers and the consumers using only the correct configurations.

[kafka-doc-producer]: https://kafka.apache.org/documentation/#producerconfigs
[kafka-doc-consumer]: https://kafka.apache.org/documentation/#consumerconfigs