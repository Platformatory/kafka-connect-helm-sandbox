# Kafka Connect Sandbox

## Features

- Single helm install command to test Kafka connectors with Strimzi
    - Installs the following -
        - Strimzi operator
        - Prometheus operator
        - Grafana, if enabled
    - Sets up a Connect Cluster and any configured connectors
- Integrated monitoring with Prometheus and Grafana or other monitoring solutions that support Prometheus remote write
    - Built in custom Grafana dashboard for connectors along with Prometheus as a Data Source
    - Enable ingress for Prometheus/Grafana

## Usage

1. Configuring connector jars

There are 2 ways to configure connector jars with the helm chart.

### Using a custom docker image (Recommended)

- Build a custom docker image using `quay.io/strimzi/kafka` as the base image. Copy the connector jars to `/opt/kafka/plugins` on the image.
- Set the `image` property of `connect` section in the values.yaml to the image name and tag of the custom image
- If you're using a private image, define `strimzi-kafka-operator.image.imagePullSecrets` and `imageCredentials`


### Using Strimzi's image build

- Refer to https://strimzi.io/docs/operators/latest/configuring.html#type-Build-reference for defining the `build` section of the `connect` property in your values.yaml.
- Define the registry authentication credentials using the `pushSecretData` section of the `connect` property in your values.yaml. The value of `pushSecretData` will be used as the data of the `registry-cred` secret.

2. Connecting to Kafka cluster

- Define the bootstrap server and the authentication required for the Kafka cluster in the `kafkaCluster` section of the values.yaml
- `bootstrapServers` property of the `kafkaCluster` section is mandatory since it is required to communicate with the kafka cluster
- If your Kafka cluster has authentication, define the `authentication` property based on https://strimzi.io/docs/operators/latest/configuring.html#type-KafkaClientAuthenticationTls-reference
- Define the authentication secret, such as password, certificates, etc., using `authenticationSecretData`
- If your Kafka cluster has tls encryption, define the `tls` property based on https://strimzi.io/docs/operators/latest/configuring.html#con-common-configuration-trusted-certificates-reference

> **Ensure your authentication secret is named `broker-auth`**

## Examples

- Custom image and Confluent Cloud brokers (SASL_PLAIN with TLS)

```yaml
connect:
  image: username/strimzi-jdbc-connect:jdbc-10.6.4

kafkaCluster:
  bootstrapServers: pkc-xxxxx.us-west4.gcp.confluent.cloud:9092
  authentication:
    type: plain
    username: XXXXXXXXXXXXXXXX
    passwordSecret:
      secretName: broker-auth
      password: password
  authenticationSecretData:
    password: YnJva2VycGFzc3dvcmQ=
  tls:
    trustedCertificates: []
```

- Defining connectors

```yaml
connectors:
  - name: jdbc-source
    spec:
      class: io.confluent.connect.jdbc.JdbcSourceConnector
      tasksMax: 3
      autoRestart:	
        enabled: true
      config:
        connection.url: jdbc:postgresql://postgres.mydomain.com:5432/kafka_connect_db
        connection.user: kafka
        connection.password: mysecurepassword
        mode: bulk
        key.converter: org.apache.kafka.connect.storage.StringConverter
        value.converter: org.apache.kafka.connect.storage.StringConverter
        value.converter.schema.registry.url: https://xxxx-yyyyy.us-east-2.aws.confluent.cloud
        table.whitelist: topic4,topic5,topic6
        value.converter.basic.auth.credentials.source: USER_INFO
        value.converter.schema.registry.basic.auth.user.info: XXXXXXXXXXXXXXXX:YYYYYYYYYYY+YYYYYYY/YYYYYYY+YYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYY
        topic.prefix: conn_
        errors.tolerance: all
        validate.not.null: false
        topic.creation.default.replication.factor: 3
        topic.creation.default.partitions: 1
```

- Private custom image for connect

```yaml
strimzi-kafka-operator:
  image:
    imagePullSecrets: private-image-pull

imageCredentials:
  registry: quay.io
  username: someone
  password: sillyness

connect:
  image: quay.io/someone/strimzi-jdbc-connect:jdbc-10.6.4
```

- Ingress for Prometheus and Grafana

```yaml
kube-prometheus-stack:
  prometheus:
    ingress:
      enabled: true
      ingressClassName: nginx
      hosts:
        - prometheus.mydomain.com
      paths:
        - /
      pathType: ImplementationSpecific
      ## TLS configuration for Prometheus Ingress
      ## Secret must be manually created in the namespace
      ##
      tls: []
        # - secretName: prometheus-general-tls
        #   hosts:
        #     - prometheus.example.com
  grafana:
    ingress:
      enabled: true
      ingressClassName: nginx
      hosts:
        - grafana.mydomain.com
      paths:
        - /
      pathType: ImplementationSpecific
      ## TLS configuration for Grafana Ingress
      ## Secret must be manually created in the namespace
      ##
      tls: []
        # - secretName: grafana-general-tls
        #   hosts:
        #     - grafana.example.com
```

- Remote monitoring tool with Prometheus Remote Write support

```yaml
kube-prometheus-stack:
  prometheus:
    prometheusSpec:
      ## ref: https://github.com/prometheus-operator/prometheus-operator/blob/main/Documentation/api.md#remotewritespec
      ## example for New Relic
      ##
      remoteWrite:
      - url: https://metric-api.newrelic.com/prometheus/v1/write?prometheus_server=kafka-connect-sandbox
        bearerToken: xxxxxxxxxxxxxxxxxxxxxxx
  ## Optionally, disable grafana since a remote monitoring solution is used
  grafana:
    enabled: false
```

