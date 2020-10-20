---
layout: post
title: Authenticated external access to a Kafka cluster in Kubernetes (part 2)
subtitle: Getting external access
share-img: /assets/img/kafka-k8s.png
tags: [kafka, kubernetes, strimzi]
comments: true
---

This time we are going to expand our example to obtain external access to the Kafka broker (from outside the Kubernetes cluster).

Communications in Kafka can be somewhat complicated, so before continuing with the article, I recommend reading a bit about it. In order to better understand the communication processes between Kafka brokers and between brokers and clients, it is recommended to read the following article:

[https://rmoff.net/2018/08/02/kafka-listeners-explained/](https://rmoff.net/2018/08/02/kafka-listeners-explained/)

The basic idea to understand is that a client connecting to a Kafka broker receives metadata about the rest of the cluster's broker during the negotiation of the communication. Armed with this information and depending on the partition assignments assigned by the broker, a client will communicate with one or more brokers directly. That is, there is no classical Master-Slave approach. Thanks to the clients connecting directly to the individual brokers, the brokers don’t need to do any forwarding of data between the clients and other brokers. That helps to reduce the amount of traffic flowing around within the cluster.

Once again Strimzi will provide us some help to getting access when we work with Kafka in Kubernetes.




### Accessing Kafka with Strimzi (from outside)

Generally, most Kubernetes clusters run on their own network which is separated from the world outside. That means things such as pod IP addresses or DNS names are not resolvable for any clients running outside the cluster. Thanks to that, it is also clear that we will need to use a separate Kafka listener for access from inside and outside of the cluster, because the advertised addresses will need to be different (again, if you are confused read the article referenced above).

Kubernetes and OpenShift have many different ways of exposing applications, such as node ports, load-balancers or routes. Strimzi supports all of these to let users find the way which suits best their use case. In this article we will obtain external access with a [Ingress Controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/).

Ingress helps us manage external access to HTTP/HTTPS services. On the one hand, Ingress resources will define the rules for routing the traffic to different services and pods. And in other hand, an Ingress controller takes care of the actual routing.



#### NGINX Ingress Controller for Kubernetes

Strimzi needs [SSL Passthrought](https://avinetworks.com/glossary/ssl-passthrough/#:~:text=Secure%20Socket%20Layer%20(SSL)%2C,server%20to%20safely%20send%20messages.&text=But%20SSL%20passthrough%20keeps%20the,travels%20through%20the%20load%20balancer.), which means our data will travel encrypted all the time instead of the more traditional approach of terminating the SSL connection on the balancer. For this and to avoid complications we will use the [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/) that is well tested by Strimzi (from the community, no to be confused with the ingress provided by Nginx Inc).



#### Note if you play this tutorial in Minikube

Minikube contains a Nginx Ingress (community version) in form of addons. We need enable it.

```bash
minikube addons enable ingress
```

As we mentioned earlier, Strimzi needs SSL Pass Through in the ingress controller. Unfortunately, this only be activate as command argument of the ingress controller. As we use the default addon in minikube, the next step is overwrite this deployment to enable SSL Pass Through. For this:

```bash
# 1) Export the current Ingress controller
kubectl -n kube-system get deployments nginx-ingress-controller -o yaml > minikube-ingress.yaml

# 2) Edit and add the lines between <NEW LINES>"
     ...
     - args:
        - /nginx-ingress-controller
        - --configmap=$(POD_NAMESPACE)/nginx-load-balancer-conf
        - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
        - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
        - --annotations-prefix=nginx.ingress.kubernetes.io
        - --report-node-internal-ip-address
        # ----- <NEW LINES> -----
        # Use minikube IP address in ingress status field
        - --report-node-internal-ip-address
        # Enable passthrough
        - --enable-ssl-passthrough
        # ----- </NEW LINES> -----
	 ...

# 3) Overwrite
kubectl apply -f minikube-ingress.yaml
```

**Important: If the cluster stop, you need to redeploy the overwrite deployment.**



### Configure our cluster

Before to change anymore, we will test the connection. [In part 1](https://7eben.dev/2020-05-18-kafka-in-k8s-part-1/), we connect to Kafka inside the cluster, for that we execute a container (as a Kafka producer) inside the cluster. But for example, if now we try to product a message from outside (with the tool [Kafkacat]( https://github.com/edenhill/kafkacat) for simplicity), we get:

```bash
tail -f /var/log/syslog | kafkacat -b zeben-cl-kafka-bootstrap -t syslog -z snappy
% Auto-selecting Producer mode (use -P or -C to override)
% ERROR: Local: Host resolution failure: zeben-cl-kafka-bootstrap:9092/bootstrap: Failed to resolve 'zeben-cl-kafka-bootstrap:9092': Temporary failure in name resolution (after 20806722ms in state INIT)
% ERROR: Local: All broker connections are down: 1/1 brokers are down : terminating
```

Of course *zeben-cl-kafka-bootstrap* is only resolved inside the cluster, but if we use the cluster IP, the result is similar.

To achieve external access, we will use one of the most used mechanisms in Kubernetes, an Ingress controller, and we will see how easy it is to use with Strimzi. To avoid editing the host or manipulate DNS service, we will use the [NIP.IO]( https://nip.io/) service. A useful free service to mock DNS resolutions. We use our cluster IP, for example, with Minikube:

```bash
sudo minikube ip
172.101.0.10
```

So our name for NIP.IO is something like:

```bash
bootstrap.172.101.0.10.nip.io or broker-X.172.101.0.10.nip.io
```

Show the minimal external access configuration:

```yaml
    ...
    listeners:
      plain: {}
      external:
        type: ingress
        configuration:
          bootstrap:
            # Example for local BOOTSTRAP (minikube) and the nip.io service
            host: bootstrap.172.101.0.10.nip.io
          brokers:
          - broker: 0
            # Example for local BROKER-0 (minikube) and the nip.io service
            host: broker-0.172.101.0.10.nip.io
          - broker: 1
            # Example for local BROKER-1 (minikube) and the nip.io service
            host: broker-1.172.101.0.10.nip.io
          - broker: 2
            # Example for local BROKER-2 (minikube) and the nip.io service
            host: broker-2.172.101.0.10.nip.io
    ...

```

Again, we apply the changes in the cluster:

```shell
kubectl apply -f kafka-cluster.yaml

# Wait for a short time and check the brokers recreation
kubectl -n kafka get pod
NAME                                        READY   STATUS    RESTARTS   AGE
strimzi-cluster-operator-77555d4b69-spxr8   1/1     Running   0          41m
zeben-cl-entity-operator-cc54d897f-zk9gd    3/3     Running   0          38m
zeben-cl-kafka-0                            2/2     Running   0          2m40s
zeben-cl-kafka-1                            2/2     Running   0          3m50s
zeben-cl-kafka-2                            2/2     Running   0          3m10s
zeben-cl-zookeeper-0                        2/2     Running   0          39m
zeben-cl-zookeeper-1                        2/2     Running   0          39m
zeben-cl-zookeeper-2                        2/2     Running   0          39m
```
Now, our kubernetes deployment is ready.


### Testing time

Well, we will execute our Kafkacat command again:

```bash
tail -f /var/log/syslog | kafkacat -b bootstrap.172.101.0.10.nip.io -t syslog -z snappy
% Auto-selecting Producer mode (use -P or -C to override)
% ERROR: Local: Broker transport failure: bootstrap.172.101.0.10.nip.io:9092/bootstrap: Connect to ipv4#172.101.0.10:9092 failed: Connection refused (after 0ms in state CONNECT)
% ERROR: Local: All broker connections are down: 1/1 brokers are down : terminating

```

Uppss, something is wrong.

Since it always uses TLS Encryption, we need to extract the cluster's certificate. Once again, Strimzi took over its generation.

```bash
kubectl -n kafka get secret zeben-cl-cluster-ca-cert -o jsonpath='{.data.ca\.crt}' | base64 -d > ca.crt
```

And then, we create a Kafkacat producer, now without errors:

```bash
tail -f /var/log/syslog | kafkacat -X ssl.ca.location=ca.crt -X security.protocol=ssl -b bootstrap.172.101.0.10.nip.io:443 -t syslog -z snappy
```

---

And finally, we create a consumer to read the generated messages:

```bash
kafkacat -X ssl.ca.location=ca.crt -X security.protocol=ssl -b bootstrap.172.101.0.10.nip.io:443 -t syslog

% Auto-selecting Consumer mode (use -P or -C to override)
Oct 19 14:34:56 perenquen systemd[1]: run-docker-runtime\x2drunc-moby-7a614d5754a21e05f503d7150c18353e544d072d328ad8bc1a6823906466629c-runc.wjXOUD.mount: Succeeded.
Oct 19 14:34:56 perenquen systemd[2265]: run-docker-runtime\x2drunc-moby-7a614d5754a21e05f503d7150c18353e544d072d328ad8bc1a6823906466629c-runc.wjXOUD.mount: Succeeded.
Oct 19 14:34:56 perenquen systemd[1088]: run-docker-runtime\x2drunc-moby-7a614d5754a21e05f503d7150c18353e544d072d328ad8bc1a6823906466629c-runc.wjXOUD.mount: Succeeded.
Oct 19 14:34:57 perenquen systemd[2265]: run-docker-runtime\x2drunc-moby-7f7edc779e3a8e38e15d79feb228decbd9c15adedb69820fc97a1988ca991424-runc.dTbhsN.mount: Succeeded.

```

And this is it, we have managed to have external access to the Kafka cluster through a Kubernetes Ingress controller. Strimzi supports various forms of external access, take a look at the documentation and take the one that best suits your needs. In the following article, we will see how to obtain this access in an authenticated way.
