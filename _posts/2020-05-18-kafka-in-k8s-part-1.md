---
layout: post
title: Authenticated external access to a Kafka cluster in Kubernetes (part 1)
subtitle: Deploying Strimzi Operator
share-img: /assets/img/kafka-k8s.png
tags: [kafka, kubernetes, strimzi]
comments: true
---

Hi guys, welcome to my new and revamped blog!. I don't want to bore you with unnecessary words in this blog. My only purpose is to share with you some experiences in the development world and I am going to open this blog with a series of posts about Kafka and Kubernetes. Here we go!


In this series of posts  I will show you an easy way to have a Kafka cluster deployed in Kubernetes and getting access outside the cluster in an authenticated way. This time I am going to assume that you have some knowledge about Kafka and Kubernetes. But I will try to explain it in the most basic way. 


We will use Strimzi for deploying Zookeeper and Kafka Broker instances. Also with Strimzi, we will configure external access to the cluster with an ingress controller. Also in our example, the communication from outside the cluster will be encrypted and authenticated, we will use mutual TLS to achieve that.


### Why Strimzi



As I said, we will use [Strimzi](https://strimzi.io/), an open source Kubernetes Operator. And, why?. First all, what is a [Kubernetes Operator](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)?. The official K8s documentation says: *"Operators are software extensions to Kubernetes that make use of [custom resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) to manage applications and their components. Operators follow Kubernetes principles, notably the [control loop](https://kubernetes.io/docs/concepts/#kubernetes-control-plane)"*



And continue says "*The Operator pattern aims to capture the key aim of a human operator who is managing a service or set of services... People who run workloads on Kubernetes often like to use automation to take care of repeatable tasks"*.*



So, in colloquial language, it's like a little robot that controls some tedious task and makes our lives easier. This is the main reason, but not the only one, I don't like [Helm](https://helm.sh/), it's not the same but I don't like it. I have my own reasons for that, but I leave it for a talk with beers.



Until now I have always used the Kafka distributions provided by Confluent. These distributions are great and have interesting tools and pretty good documentation. However its Kubernetes operator is very limited (and also uses Helm). I will continue to use their distributions in my developments but not this time.



So Strimzi is my choice.  The documentation is excellent and of course it is an open source project. They have recently updated their website and now it looks great. Take a look.



### Choice your cluster



In order not to complicate this post too much, I will use Minikube to deploy our Kubernetes cluster. I will try not to deploy too many replicas but keep in mind that you need a decent machine to run this example. If you prefer, you can use your favorite provider of Kubernetes. I use AWS EKS in my work and everything shown here obviously works fine, but also in other providers like GKE, AKS, DigitalOcean or for example, with Red Hat OpenShift.



## Deploy a Strimzi operator



1. First, we create a k8s namespace for simply.

   ```shell
   kubectl create namespace kafka
   ```

2. Prepare the Strimzi operator to use our namespace.

   ```shell
wget https://github.com/strimzi/strimzi-kafka-operator/releases/download/0.16.2/strimzi-cluster-operator-0.16.2.yaml
   
   sed 's/namespace: .*/namespace: kafka/' strimzi-cluster-operator-0.16.2.yaml > strimzi-cluster-operator-0.16.2-kafka.yaml
   ```
   
3. Deploy the operator

   ```shell
 kubectl apply -f strimzi-cluster-operator-0.16.2-kafka.yaml -n kafka
    
    # Or if you prefer download the splitted version of manifest
    kubectl apply -f strimzi/cluster-operator
    kubectl apply -f strimzi/topic-operator/
    kubectl apply -f strimzi/user-operator/
   ```

{:  .box-note }

**Note:** 
May be you prefer to apply K8s deployments without using *kubectl* directly, for example with *Gitops* tools. If you have a cluster and want to maintain a namespace for Kafka deployments, we should complete the definition of namespaces in manifests to avoid creating objects in the default namespace. An easy way for this, is to use [Kustomize](https://kustomize.io/).


## Deploy the Kafka Cluster


At the final of this post series, we will have a Kafka cluster with the following set of characteristics:

- **Number of Zookeeper instances:** 3
- **Number of Broker instances:** 3
- **External access to cluster:** Yes, with a Nginx Ingress controller.
- **Encryption:** Yes, for external connections
- **Authentication:** Yes, mutual TLS for external connections


But for now, just create a cluster with the first two points requirements of the list. We create a Yaml (`kafka-cluster.yaml)` with the following content:

```yaml
apiVersion: kafka.strimzi.io/v1beta1
kind: Kafka
metadata:
  name: zeben-cl
  namespace: kafka  
spec:
  kafka:
    version: 2.4.0
    replicas: 3
    listeners:
      plain: {}
    config:
      offsets.topic.replication.factor: 1
      transaction.state.log.replication.factor: 1
      transaction.state.log.min.isr: 1
      log.message.format.version: "2.4"
    storage:
      type: persistent-claim
      size: 100Mi
      deleteClaim: false
  zookeeper:
    replicas: 3
    storage:
      type: persistent-claim
      size: 100Mi
      deleteClaim: false      
  entityOperator:
    topicOperator: {}
    userOperator: {}
```


And then apply it to the cluster.

```shell
kubectl apply -f kafka-cluster.yaml -n kafka
```


And thats all, if all was fine, you have a Kafka cluster ready. Let check it. 



## Testing time


We have 8 pods inside the Kafka namespace. Two for our Strimzi cluster, three zookeepers and three Kafka brokers.

```shell
$ kubectl -n kafka get pod
NAME                                        READY   STATUS    RESTARTS   AGE
strimzi-cluster-operator-77555d4b69-x64k7   1/1     Running   0          4m35s
zeben-cl-entity-operator-cc54d897f-gm7xm    3/3     Running   0          3m3s
zeben-cl-kafka-0                            2/2     Running   0          3m27s
zeben-cl-kafka-1                            2/2     Running   0          3m27s
zeben-cl-kafka-2                            2/2     Running   0          3m27s
zeben-cl-zookeeper-0                        2/2     Running   0          3m56s
zeben-cl-zookeeper-1                        2/2     Running   0          3m56s
zeben-cl-zookeeper-2                        2/2     Running   0          3m56s
```


Once the cluster is running, you can run a simple producer to send messages to a Kafka topic (the topic will be automatically created):

```shell
kubectl -n kafka run kafka-producer -ti --image=strimzi/kafka:0.16.2-kafka-2.4.0 --rm=true --restart=Never -- bin/kafka-console-producer.sh --broker-list zeben-cl-kafka-bootstrap:9092 --topic example-topic
```


Enter some text:
```shell
$ kubectl -n kafka run kafka-producer -ti --image=strimzi/kafka:0.16.2-kafka-2.4.0 --rm=true --restart=Never -- bin/kafka-console-producer.sh --broker-list zeben-cl-kafka-bootstrap:9092 --topic example-topic
If you don't see a command prompt, try pressing enter.
>first message
>two message
>three message
>
```


And to receive them in a different terminal you can run:

```shell
kubectl -n kafka run kafka-consumer -ti --image=strimzi/kafka:0.17.0-kafka-2.4.0 --rm=true --restart=Never -- bin/kafka-console-consumer.sh --bootstrap-server zeben-cl-kafka-bootstrap:9092 --topic example-topic --from-beginning
```


And see the result:

```shell
$ kubectl -n kafka run kafka-consumer -ti --image=strimzi/kafka:0.17.0-kafka-2.4.0 --rm=true --restart=Never -- bin/kafka-console-consumer.sh --bootstrap-server zeben-cl-kafka-bootstrap:9092 --topic example-topic --from-beginning
If you don't see a command prompt, try pressing enter.
first message
two message
three message
```


That's all for now. Basically we have replicated the "Getting started"  of Strimzi documentation. We will continue to complicate our example in the following posts.

