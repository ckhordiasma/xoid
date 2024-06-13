---
layout: post
title: "Kubernetes course notes"
date: 2024-04-18
---

This is a collection of miscellaneous notes I took while taking a Udemy course on Kubernetes.

## Useful trick for testing services:

1. kubectl describe service xxxx
2. get the endpoint:port for this service
3. run a busybox pod/container and connect to it
4. from the busybox shell, telnet the endpoint:port and run GET /



# scaling pods

to scale pods, you can use a replication controller as such:

```
apiVersion: v1
kind: ReplicationController
metadata:
  name: helloworld-replicas
spec:
  replicas: 2
  selector:
    app: <<APP LABEL>>
  template:
    <<POD METADATA AND SPEC YAML HERE HERE, INCLUDING APP LABEL>>
```

and just run `kubectl apply -f <filename>`. If you later decide that you want MORE, you can run

```
kubectl scale --replicas=4 -f <filename>
# or (rc is abbreviation for replicationcontroller)
kubectl scale --replicas=4 rc/helloworld-replicas
```

to start making more copies

# replica sets

next generation replication generator. Replica Sets can do selection based on filtering on a set of values instead of only strict equalities in the rep controller

deployments use replica sets instead of replication controllers.

# deployments

allows for deployment and app updates. can do rolling updates with zero downtime! also can do rollbacks, and can pause and resume.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: helloworld-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: <APP LABEL NAME>
  template:
    << POD METADATA AND SPEC YAML HERE INCLUDING APP LABEL>>
```

Some useful deploy related commands:

```
kubectl get deploy
kubectl get rs 
kubectl get pods --show-labels
kubectl rollout status deploy/<NAME>

# reverting to previous deployments
kubectl rollout history deploy<name>
kubectl rollout undo deploy/<name>
kubectl rollout undo deploy/<name> --to-revision=n

# changing specific images in pod
kubectl set image deploy/<NAME> CONTAINER_NAME=NEW_IMAGE_NAME_AND_TAG
# can also add a --record tag to show notes in deployment history during deployments

# editing deployment directly
kubectl edit deploy/<NAME>

```

# services

exposes pod/deployments to the world or other pods.

- ClusterIP - virtual ip address that is internal to the cluster only (default)
- NodePort - a port that is the same on each node that is also reachable external to cluster
- LoadBalancer - something created by the service provider that will translate external traffic into the correct NodePorts 
- Ingress - some kind of alternative to LB and NP - can easily expose services...?

# Ingress controller

1. internet requests hit the ingress controller service
2. service sends requests to the ingress controller pod/containers
3. ingress controller pod figures out what actual service needs to be served based on the request
4. ingress controller pod talks to actual service pods to get the served data

when i was trying to do an ingress controller I ran into an issue where apparently I needed to include a metadata annotation in my ingress yaml config.

## benefits of ingress controllers

- only need one Load Balancer on the AWS cloud provider side to pass traffic to ingress
- one cloud provider Load Balancer = less costs 
- mainly only good for http/s applications

# External DNS

- a tool to integrate k8s with external (i.e. cloud) DNS
- for every hostname used in ingress, External DNS will interface with cloud to create a new record
- Google Cloud, Amazon, Azure, Cloudflare, DigitalOcean are built in
- usually a pod running in the cluster that reads ingress rules 

went through a lot of struggles to get this set up. had to adjust a bunch of things in kops and stuff

# StatefulSet vs Replica Sets

StatefulSets end in -0, -1, -2 instead of a random string. It's useful for when you have volumes that attach to the pod containers, or if you want to map DNS to your pods directly since you know the pod names will not change.

# metrics server for autoscaling

was able to `kubectl apply` from the metrics server github repo. once that was installed and the relevant container in the `kube-system` namespace started, i could run commands such as: `kubectl top pods` and `kubectl top nodes`
