# Deploying Kafka with Strimzi

To get started, you will have to download Strimzi. The latest release can be found [here](http://strimzi.io/downloads/)

Decompress and extract the Strimzi release tarball

```
curl https://github.com/strimzi/strimzi-kafka-operator/releases/download/0.8.2/strimzi-0.8.2.tar.gz -O
tar xzf strimzi-0.8.2.tar.gz
```



```
oc login -u system:admin
oc apply -f install/cluster-operator
oc apply -f examples/templates/cluster-operator
oc apply -f examples/kafka/kafka-ephemeral.yaml
```

Deploy basic File Sink Kafka Connect Cluster

```
oc apply -f examples/kafka-connect/kafka-connect.yaml
```
