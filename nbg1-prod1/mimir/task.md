Deploy Grafana Mimir with the mimir-distributed Helm chart. https://github.com/grafana/mimir/blob/main/operations/helm/charts/mimir-distributed

Install the latest version of the chart.

Install with "ingest storage" enabled. This requires a kafka cluster. Deploy Kafka
using the Strizi kafka operator (already installed on the cluster).

ingest storage details: https://grafana.com/docs/mimir/latest/get-started/about-grafana-mimir-architecture/about-ingest-storage-architecture/

The cluster is not that big, so this deployment should be small but still highly available.

There is an example values file for a small deployment here: https://github.com/grafana/mimir/blob/main/operations/helm/charts/mimir-distributed/small.yaml

Use the bucket "bc4-nbg1-prod1-mimir" on the S3-compatible object storage for long-term storage. See Loki deployment for an example.

Use the same credentials as Loki to access the S3-compatible object storage - deployed as an external secret.

create a namespace resource. enable istio ambient mesh injection for the namespace.

check mimir kafka requirements https://grafana.com/docs/mimir/latest/configure/configure-kafka-backend/

do not use zookeeper with kafka. use kraft. docs https://strimzi.io/docs/operators/latest/deploying
some examples here https://github.com/strimzi/strimzi-kafka-operator/tree/0.49.1/examples/kafka