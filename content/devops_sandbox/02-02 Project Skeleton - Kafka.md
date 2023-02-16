---
title: 02-02 Project Skeleton - Kafka
fileName: 02-02 Project Skeleton - Kafka
tags:
- kafka
- statefulset
- k8s
categories:
- 
date: 2023-02-14
lastMod: 2023-02-16
ShowPostNavLinks: true
ShowBreadCrumbs: true
ShowReadingTime: true
---
# Kafka

What is one of the component I often use at work? You guessed it, [Kafka](https://kafka.apache.org/). For those who don't know, Kafka can be seen as a messaging system (the exact term is streaming platform), where some will publish messages (Producers) while others will process them (Consumers). It is made to scale, so it is obviously overkill for a personal project, but the goal was to experiment with it. Also, I will not delve here into the details of Kafka now (maybe when I tackle [Observability](https://en.wikipedia.org/wiki/Observability)?)

Currently, you need 2 type of components to leverage Kafka:

  + the Brokers that will handle all the actual streaming of messages

  + [Zookeeper](https://zookeeper.apache.org/) that will coordinate the brokers

Having 2 components means 2 configurations to get right, and even more occasions to make errors. As such, there is an ongoing effort to get rid of Zookeeper, and only have the brokers. This new approach is called [KRaft](https://developer.confluent.io/learn/kraft/), and the brokers will now communicate through a dedicated topic to share information, instead of going through Zookeeper. The only issue is that it is still a work in progress (02/2023), so it should not be used in production yet. However, this should not stop me for my experimentations.

## Service

`service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kafka-svc
  labels:
    app: kafka-app	
spec:
  clusterIP: None
  ports:
    - name: '9092'
      port: 9092
      protocol: TCP
      targetPort: 9092
  selector:
    app: kafka-app
```

You can see with `clusterIP: None` that no IP is assigned to the service. It is called a **Headless Service**, and is used when there is no need for load balancing, and the access to the service will be done through its name, e.g. `kafka-svc:9092` in this specific case.

## Statefulset

`statefulset.yaml`

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kafka
  labels:
    app: kafka-app
spec:
  serviceName: kafka-svc
  replicas: 1
  selector:
    matchLabels:
      app: kafka-app
  template:
    metadata:
      labels:
        app: kafka-app
    spec:
      containers:
        - name: kafka-container
          image: bashj79/kafka-kraft:latest
          ports:
            - containerPort: 9092
            - containerPort: 9093
          env:
            - name: KAFKA_ADVERTISED_LISTENERS
              value: PLAINTEXT://kafka-svc:9092,CONTROLLER://kafka-svc:9093
            - name: KAFKA_LISTENER_SECURITY_PROTOCOL_MAP
              value: PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT
          volumeMounts:
            - name: data
              mountPath: /mnt/kafka
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes:
          - "ReadWriteOnce"
        resources:
          requests:
            storage: "1Gi"
```

This is my first time with [Statefulsets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)! It can be seen as a Deployment that will automatically create a [PersistentVolume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) and the corresponding PersistentVolumeClaim. The interesting thing with Statefulsets is that they are "sticky": if a pod needs to be redeployed, it will have the same name (e.g. `name-0`, `name-1`, ...) and will bind to the same volume (e.g. `name-0` will always bind to `volume-0`).

The image I've used (`bashj79/kafka-kraft:latest`) is a Kafka image enabling KRaft with a default configuration. My guess is that the biggest difference with usual Kafka images lies in the KRaft-specific configuration, where the following fields (among others) are filled:

```
node.id=1
process.roles=broker,controller
controller.listener.names=CONTROLLER
controller.quorum.voters=1@localhost:9093
```

I initially thought that this was enough for redundancy, and increasing the number of replicas would be enough. But I'll detail it in the following section...

## Checks

As a programmer, I've learned that "testing is doubting" and bad things happen only when you are looking. So I started looking.

### Network checks

Once applied (`kubectl apply -k .`) I can see my service/pod happily running! (I aliased `kubectl` as `k`)

```bash
$ k get service
NAME           TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
kafka-svc      ClusterIP   None         <none>        9092/TCP   46m
$ k get pods
NAME           READY   STATUS    RESTARTS       AGE
kafka-0        1/1     Running   0              6m31s
```

First I was not sure how to communicate with my service: it does not have an IP, and what is its name? For this I will use `nslookup` from a pod:

```bash
$ k run dig --rm -ti --image arunvelsriram/utils -- bash
utils@dig:~$ nslookup kafka-svc
Server:		10.96.0.10
Address:	10.96.0.10#53

Name:	kafka-svc.default.svc.cluster.local
Address: 10.244.0.9
```

So using the service name `kafka-svc` seems to be enough! Also, it returned me a [Fully Qualified Domain Name (FQDN)](https://fr.wikipedia.org/wiki/Fully_qualified_domain_name) (`kafka-svc.default.svc.cluster.local`) with it's associated IP. This corresponds to the single pod I've deployed, and sure enough prepending the pod name (e.g. `kafka-0`) to the FQDN will yield:

```bash
utils@dig:~$ dig kafka-0.kafka-svc.default.svc.cluster.local
[...]
;; ANSWER SECTION:
kafka-0.kafka-svc.default.svc.cluster.local. 30	IN A 10.244.0.9
```

It's the same IP we've seen above!

With multiple replicas, `nslookup kafka-svc` will return multiple results, one for each replica. It's just that when connecting to `kafka-svc`, sometimes it will be `kafka-0.kafka-svc.default.svc.cluster.local`, or `kafka-1.kafka-svc.default.svc.cluster.local`, or `kafka-2.kafka-svc.default.svc.cluster.local`. This, coupled with a bad understanding of how the brokers were syncing, led me to several headaches...

### Kafka checks

So far everything was fine, and I boldly decided to test if I can publish/consume on my kafka topics. To do so I used to following commands:

```bash
$ k run kafka-client --rm -ti --image bitnami/kafka:3.1.0 -- bash
# if already running
$ k exec kafka-client -ti -- bash

# Consume
$ kafka-console-consumer.sh --topic first_topic --bootstrap-server kafka-svc:9092
# Produce
I have no name!@kafka-client:/$ kafka-console-producer.sh --topic first_topic --bootstrap-server kafka-svc:9092
```

As far as I'm concerned, it seems to be working (producer on left, consumer on right):

![image.png](/assets/image_1676233158650_0.png)

However, increasing the number of replicas (in the `statefulset.yaml` file) seems to break things. Indeed, sometimes my consumers will straight up ignore the producers, while sometimes talking to each other nicely. I was puzzled, until I reduced the number of replicas back to 1, where everything was working nicely.

That's when I realized my issue with the configuration, and how naive I was thinking that increasing the replicas will just magically work. The default configuration was always assigning the same node id, and the voter was always the same (`1@localhost:9093`). What I need is probably a node id aligned with the Statefulset's id (e.g. `kafka-0` should have `node.id=0`) and a list of voters with the proper FQDN (`0@kafka-0.kafka-svc.default.svc.cluster.local:9093,1@kafka-1.kafka-svc.default.svc.cluster.local:9093,2@kafka-2.kafka-svc.default.svc.cluster.local:9093` for 3 replicas).

However, this is beyond the current scope (I just want to make a skeleton), and I'll probably need to dig into making my own container image, generate a configuration file specific to a pod (maybe just the `node.id`?), while making sure all is working with whatever number of replicas I want. I'll probably tackle those problems specifically when looking into resiliency.

## Takeaways

Statefulsets are cool: they are Deployments with sticky volumes.

Resiliency is not magical, it is done on the software level with some elbow grease.

Testing is doubting.

Just kidding: the process is frustrating, but it is a good way to check the state of my understanding.
