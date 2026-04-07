+++
title = 'Spinkube Tour'
date = 2026-03-31T16:56:21+08:00
draft = true
+++

> Interested in Spinkube recently, this article basically marks what I learned during reading docs.

## Docker Proxy Configuration

> 光和代理斗智斗勇了

`/etc/docker/daemon.json` : using https://1ms.run/

`/etc/systemd/system/docker.service.d/proxy.conf` : daemon proxy configuration

```bash
[Service]
Environment="HTTP_PROXY=http://127.0.0.1:10809"
Environment="HTTPS_PROXY=http://127.0.0.1:10809"
Environment="NO_PROXY=localhost,127.0.0.1"
```

`/etc/privoxy/config` : listen at SOCKS5(10808)

```bash
forward-socks5t / 127.0.0.1:10808 .
listen-address 127.0.0.1:10809
```

## Quickstart

https://www.spinkube.dev/docs/install/quickstart/

> this article guides how to set up a new k8s cluster, install the SpinKube and deploy your first Spin application.

Mainly use k3d to show functions, for boosting pros of wasm and personal use.

 ### Setup Kubernetes Cluster

```bash
k3d cluster create wasm-cluster \
  --image ghcr.io/spinframework/containerd-shim-spin/k3d:v0.23.0 \
  --port "8081:80@loadbalancer" \
  --agents 2

```

+ create a cluster named wasm-cluster
+ using the k3d image configured by Spin team, replacing containerd-shim to the one that supports wasm (which is originally **runc**)

+ export loadbalancer port
+ setup two client

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.3/cert-manager.yaml
kubectl wait --for=condition=available --timeout=300s deployment/cert-manager-webhook -n cert-manager
```

+ add cert-manager

```bash
kubectl apply -f https://github.com/spinframework/spin-operator/releases/download/v0.6.1/spin-operator.runtime-class.yaml
kubectl apply -f https://github.com/spinframework/spin-operator/releases/download/v0.6.1/spin-operator.crds.yaml
```

+ add runtime-class
+ add crds

### Deploy the Spin Operator

```bash
# Install Spin Operator with Helm
helm upgrade --install spin-operator \
  --namespace spin-operator \
  --create-namespace \
  --version 0.6.1 \
  --wait \
  oci://ghcr.io/spinframework/charts/spin-operator
```

```bash
kubectl apply -f https://github.com/spinframework/spin-operator/releases/download/v0.6.1/spin-operator.shim-executor.yaml
```

### Run the Sample Application

```bash
kubectl apply -f https://raw.githubusercontent.com/spinframework/spin-operator/main/config/samples/simple.yaml
```

+ create spin app

```bash
kubectl port-forward svc/simple-spinapp 8083:80

Forwarding from 127.0.0.1:8083 -> 80
Forwarding from [::1]:8083 -> 80
```

+ forward clusterip port to localhost:8083

```bash
curl localhost:8083/hello

Hello world from Spin!% 
```

You can see the result.

