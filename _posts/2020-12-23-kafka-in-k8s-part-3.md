---
layout: post
title: Authenticated external access to a Kafka cluster in Kubernetes (part 3)
subtitle: Authenticated access
image: /assets/img/kafka-k8s.png
share-img: /assets/img/kafka-k8s.png
tags: [kafka, kubernetes, strimzi]
comments: true
---

Hi again, we will finish the last part of this series of posts by securing our Kafka connection for authenticated external access.

There are different ways to implement authentication and authorization with Kafka, and again Strimzi supports different mechanism to achieve our goal. But in this article, we will only focus on securing external access. Internal communications, for example, intra-broker communications, will remain unauthenticated. If you have looked at the cluster configuration `yaml` you will see that the internal listener is marked as "`plain: {}`". And in the same way, we will focus on the authentication part and not the authorization part.



### About the authentication

If you remember in the [previous blog entry,](../2020-10-20-kafka-in-k8s-part-2) we established a communication channel using TLS through which communications are encrypted. So, we will use this channel to add the convenient authentication mechanism. 

In this example we will use "Mutual TLS" authentication as the authentication mechanism in Kafka's "listeners". Strimzi supports other froms of authentication, please refer to the official documentation to explore it.

This two-way authentication occurs when the server and the client present certificates, i.e, with this authentication the broker authenticates the client and the client authenticates the broker. Certificate manipulation can be somewhat tedious but again we delegate in the Strimzi operator to create these certificates and perform maintenance operations on them. It is impressive and allow us to keep  our example  simple.



### Configure our previous cluster

We need to modify our previous cluster. On the one hand we add "TLS" authentication in our external listener. Something like:

```yaml
...
  kafka:
    version: 2.4.0
    replicas: 3
    listeners:
      plain: {}
      external:
        type: ingress
        authentication:
          type: tls
```

In other hand, we configure some parameters about the generation of our certificates. For example, we enable cluster and client certificate generation and their validity time. In the same way, we define when the automatic renewal of certificates should be carried out.

```yaml
...
  clusterCa:
    generateCertificateAuthority: true
    # 5 years
    validityDays: 1825
    # certificate renewal before the old CA certificates expire
    renewalDays: 31
  clientsCa:
    generateCertificateAuthority: true
    # 5 years
    validityDays: 1825
    # Days certificate renewal before the old CA certificates expire
    renewalDays: 31  
...
```

Please note, that the operator will need to perform a rolling restart if a CA certificate that it manages is close to expiry. But this is not a problem if the cluster is well dimensioned. However, it is possible to define maintenance windows where the operator will perform the rolling start.

You can get the full example in my [GitHub repository](https://github.com/7eben/redoubt-7eben-examples). As always, we apply our new configuration:

```bash
kubectl -n kafka apply -f kafka-cluster.yaml
```

After a few minutes all brokers will be restarted.

Now, we need to create a user to use Kafka:

```
kubectl apply -f my-user.yaml -n kafka
```

And Strimzi will generate the needles certifcates.


### Testing time

Well, we will execute our Kafkacat command again:

```bash
tail -f /var/log/syslog | kafkacat -b bootstrap.172.101.0.10.nip.io -t syslog -z snappy
% Auto-selecting Producer mode (use -P or -C to override)
% ERROR: Local: SSL error: ssl://bootstrap.192.168.1.37.nip.io:443/bootstrap: SSL handshake failed: ../ssl/statem/statem_clnt.c:393: error:141A10F4:SSL routines:ossl_statem_client_read_transition:unexpected message: : client authentication might be required (see broker log) (after 16ms in state CONNECT)
% ERROR: Local: All broker connections are down: 1/1 brokers are down : terminating

```

As you may have noticed, now the exception has changed compared to the one we saw in the previous post. Now we cannot connect to the cluster because the handshake authentication has failed.

Since it always uses TLS authentication, we need to extract the user's certificate and the password certificate. 

```bash
# Certificate
kubectl -n kafka get secret my-user -o jsonpath='{.data.user\.crt}' | base64 -d > user.crt

# Key
kubectl -n kafka get secret my-user -o jsonpath='{.data.user\.key}' | base64 -d > user.key

# Key password
kubectl -n kafka get secret my-user -o jsonpath='{.data.user\.password}' | base64 -d > user.password

```

Strimzi also offers us the user certificate in *p12* format that is more suitable to use for example in any Kafka connector. We will focus on making it work with the Kafkacat tool.

After that we need to configure our Kafkacat arguments to use the correct certificates and execute it, now without errors:

```bash
tail -f /var/log/syslog | \
kafkacat \
    -b bootstrap.192.168.1.37.nip.io:443 \
    -X security.protocol=SSL \
    -X ssl.key.location=user.key \
    -X ssl.key.password=$(cat user.password) \
    -X ssl.certificate.location=user.crt \
    -X ssl.ca.location=ca.crt \
    -t syslog -z snappy
```


And finally, we create a consumer to read the generated messages:

```bash
kafkacat \
    -b bootstrap.192.168.1.37.nip.io:443 \
    -X security.protocol=SSL \
    -X ssl.key.location=user.key \
    -X ssl.key.password=dNh0Qu60KtbX \
    -X ssl.certificate.location=user.crt \
    -X ssl.ca.location=ca.crt \
	-t syslog
```

And the output:

```
% Auto-selecting Consumer mode (use -P or -C to override)
Dec 25 13:31:56 perenquen systemd[1124]: run-docker-runtime\x2drunc-moby-d5f665aa49917bd0226ec99516f532b9dc5c7f09d87b5d738dc1d3ea1f37e122-runc.ZCw5re.mount: Succeeded.
Dec 25 13:31:57 perenquen systemd[1727]: run-docker-runtime\x2drunc-moby-dcf612d411e404b6a937ae602c50068648815415974f4fb32cc857b47744291a-runc.UB90VJ.mount: Succeeded.
Dec 25 13:31:57 perenquen systemd[1124]: run-docker-runtime\x2drunc-moby-dcf612d411e404b6a937ae602c50068648815415974f4fb32cc857b47744291a-runc.UB90VJ.mount: Succeeded.
Dec 25 13:31:57 perenquen systemd[1]: run-docker-runtime\x2drunc-moby-dcf612d411e404b6a937ae602c50068648815415974f4fb32cc857b47744291a-runc.UB90VJ.mount: Succeeded.
Dec 25 13:31:58 perenquen systemd[1727]: run-docker-runtime\x2drunc-moby-d5f665aa49917bd0226ec99516f532b9dc5c7f09d87b5d738dc1d3ea1f37e122-runc.qOEacx.mount: Succeeded.
Dec 25 13:31:58 perenquen systemd[1124]: run-docker-runtime\x2drunc-moby-d5f665aa49917bd0226ec99516f532b9dc5c7f09d87b5d738dc1d3ea1f37e122-runc.qOEacx.mount: Succeeded.
Dec 25 13:31:58 perenquen systemd[1]: run-docker-runtime\x2drunc-moby-d5f665aa49917bd0226ec99516f532b9dc5c7f09d87b5d738dc1d3ea1f37e122-runc.qOEacx.mount: Succeeded.
Dec 25 13:31:58 perenquen systemd[1727]: run-docker-runtime\x2drunc-moby-428d6b8a6a6b1da4fa14dea85b3db7f8bbed9d465c1ce9448d05e327d981e90a-runc.UykRG5.mount: Succeeded.
```

Right, all works perfect!

And that ends the three-post series about Kafka on K8s. Strimizi is constantly evolving and every day it has more and better options. This has only been a kind of "hello world" to achieve our initial goal of having an external and authenticated access to a Kafka cluster deployed in Kubernetes.

If you want to have a kafka deployment on Kubernetes, I recommend it over the rest of the solutions that the market offers. What do you think?