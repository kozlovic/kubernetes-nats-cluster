# kubernetes-nats-cluster
NATS cluster on top of Kubernetes made easy.

[![Docker Repository on Quay](https://quay.io/repository/pires/docker-nats/status "Docker Repository on Quay")](https://quay.io/repository/pires/docker-nats)

## Pre-requisites

* Kubernetes cluster (tested with v1.2.4 on top of [Vagrant + CoreOS](https://github.com/pires/kubernetes-vagrant-coreos-cluster))
* GKE 1.1.x and 1.2.x
* `kubectl` configured to access your cluster master API Server
* OpenSSL for TLS certificate generation.

## How I built the image

First, one needs to build `gnatsd` that supports the topology gossiping released with version `0.8.0`..
```
cd $GOPATH/src/github.com/nats-io/gnatsd
git pull --rebase origin master
git co tags/v0.8.0
GOARCH=amd64 GOOS=linux go build
```

Then, I copied the resulting binary to this repository `artifacts` folder, committed and proceeded to push a new tag that will trigger an automatic build:
```
git tag 0.8.0
git push
git push --tags
```

## Deploy

```
kubectl create -f svc-nats.yaml
kubectl create -f rc-nats.yaml
```

## Scale

```
kubectl scale rc/nats --replicas=3
```

Did it work?

```
$ kubectl get svc,rc,pods
NAME         CLUSTER_IP   EXTERNAL_IP   PORT(S)                      SELECTOR         AGE
kubernetes   10.100.0.1   <none>        443/TCP                      <none>           58m
nats         None         <none>        4222/TCP,6222/TCP,8222/TCP   component=nats   23m
CONTROLLER   CONTAINER(S)   IMAGE(S)                            SELECTOR         REPLICAS   AGE
nats         nats           quay.io/pires/docker-nats:0.8.0   component=nats   3          23m
NAME         READY     STATUS    RESTARTS   AGE
nats-c3eu2   1/1       Running   0          23m
nats-ruu5q   1/1       Running   0          21m
nats-dke71   1/1       Running   0          21m
```

## Access the service

*Don't forget* that services in Kubernetes are only acessible from containers in the cluster.

In this case, we're using a [`headless service`](http://kubernetes.io/v1.1/docs/user-guide/services.html#headless-services).

Just point your client apps to:
```
nats:4222
```

## TLS

First, we need to generate a valid TLS certificate:
```
openssl genrsa -out ca-key.pem 2048
openssl req -x509 -new -nodes -key ca-key.pem -days 10000 -out ca.pem -subj "/CN=kube-ca"
openssl genrsa -out nats-key.pem 2048
openssl req -new -key nats-key.pem -out nats.csr -subj "/CN=kube-nats" -config ssl.cnf
openssl req -new -key nats-key.pem -out nats.csr -subj "/CN=kube-nats" -config ssl.cnf
openssl x509 -req -in nats.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out nats.pem -days 3650 -extensions v3_req -extfile ssl.cnf
```

Now, it's time to create a Kubernetes secret to store the certificate files:
```
kubectl create secret generic tls-nats --from-file nats.pem --from-file nats-key.pem
```

Finally, deploy a secured NATS cluster:
```
kubectl create -f rc-nats-tls.yaml
kubectl scale rc/nats --replicas=3
```
