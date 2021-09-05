# linkerd-test-camilo
files used for demo on Linkerd  traffic control and deployment control

# Introduction to Service Mesh

Using examples and codes from channel "That devops guy"
## https://github.com/marcel-dempers/docker-development-youtube-series

# Service Mesh Guides

## Introduction to Linkerd

Getting started with Linkerd

Read Me:  [readme](./linkerd/README.md)

Video :point_down: <br/>

<a href="https://youtu.be/Hc-XFPHDDk4" title="Introduction to Linkerd for beginners | a Service Mesh"><img src="https://i.ytimg.com/vi/Hc-XFPHDDk4/hqdefault.jpg" width="45%" height="45%" alt="Linkerd" /></a>

## ===== demo part =====

# Introduction to Linkerd

## We need a Kubernetes cluster

Option 1: local cluster using [kind](https://kind.sigs.k8s.io/docs/user/quick-start/)
Option 2: Cloud cluster hosted in GCP or AWS

following examples demonstrate with a local cluster

```
kind create cluster --name linkerd --image kindest/node:v1.19.1
```

## Deploy our microservices (Video catalog)

```
# ingress controller
kubectl create ns ingress-nginx
kubectl apply -f kubernetes/servicemesh/applications/ingress-nginx/

# applications
kubectl apply -f kubernetes/servicemesh/applications/playlists-api/
kubectl apply -f kubernetes/servicemesh/applications/playlists-db/
kubectl apply -f kubernetes/servicemesh/applications/videos-web/
kubectl apply -f kubernetes/servicemesh/applications/videos-api/
kubectl apply -f kubernetes/servicemesh/applications/videos-db/
```

## Make sure our applications are running 

```
kubectl get pods
NAME                            READY   STATUS    RESTARTS   AGE  
playlists-api-d7f64c9c6-rfhdg   1/1     Running   0          2m19s
playlists-db-67d75dc7f4-p8wk5   1/1     Running   0          2m19s
videos-api-7769dfc56b-fsqsr     1/1     Running   0          2m18s
videos-db-74576d7c7d-5ljdh      1/1     Running   0          2m18s
videos-web-598c76f8f-chhgm      1/1     Running   0          100s 

```

## Make sure our ingress controller is running

```
kubectl -n ingress-nginx get pods
NAME                                        READY   STATUS    RESTARTS   AGE  
nginx-ingress-controller-6fbb446cff-8fwxz   1/1     Running   0          2m38s
nginx-ingress-controller-6fbb446cff-zbw7x   1/1     Running   0          2m38s

```

We'll need a fake DNS name `servicemesh.demo` <br/>

Option 1: if running local on Windows, add the following entry in the hosts file (`C:\Windows\System32\drivers\etc\hosts`) file: <br/>

```
127.0.0.1  servicemesh.demo

```

Option 2: if running on a Cloud cluster, add the External IP accessible to the service in the hosts file (`C:\Windows\System32\drivers\etc\hosts`) file: <br/>

```
External_Service_IP servicemesh.demo

```


## Let's access our applications via Ingress, if service is ClusterIP

```
kubectl -n ingress-nginx port-forward deploy/nginx-ingress-controller 80
```

## Access our application in the browser
## Make sure you have added the hostname to the host file

We should be able to access our site under `http://servicemesh.demo/home/`

<br/>
<hr/>

# Getting Started with Linkerd

On containers <br/>

## Get a container to work in
<br/>
Run a small `alpine linux` container where we can install and play with `linkerd`: <br/>

```
docker run -it --rm -v ${HOME}:/root/ -v ${PWD}:/work -w /work --net host alpine sh

##container ephemere using rm 

# install curl & kubectl
apk add --no-cache curl nano
curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
chmod +x ./kubectl
mv ./kubectl /usr/local/bin/kubectl
export KUBE_EDITOR="nano"

#test cluster access:
/work # kubectl get nodes
NAME                    STATUS   ROLES    AGE   VERSION
linkerd-control-plane   Ready    master   26m   v1.19.1

```

## Linkerd CLI

install <br/>



instructions on linkerd page [linkerd install](https://linkerd.io/2.10/getting-started/#step-1-install-the-cli)

```
curl -sL run.linkerd.io/install | sh

linkerd --help
```

## Pre flight checks

Linkerd has a great capability to check compatibility with the target cluster <br/>

```
linkerd check --pre

```

## if Get the YAML

```
linkerd install > ./kubernetes/servicemesh/linkerd/manifest/linkerd-mainifest.yaml
```

## Install Linkerd

```
linkerd install | kubectl apply -f -
```

Let's wait until all components are running

```
watch kubectl -n linkerd get pods
kubectl -n linkerd get svc
```

## Do a final check

```
linkerd check
```

## The dashboard

Let's access the `linkerd` dashboard via `port-forward`

```
kubectl -n linkerd port-forward svc/linkerd-web 8084
```

# Mesh the demo application, the video catalog services

There are 2 ways to mesh:

1) We can add an annotation to your deployment to persist the mesh if our YAML is part of a GitOps flow:
This is a more permanent solution:

```
  template:
    metadata:
      annotations:
        linkerd.io/inject: enabled
```

2) Or inject `linkerd` on the fly:
This may only be temporary as your CI/CD system may roll out the previous YAML

```
kubectl get deploy
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
playlists-api   1/1     1            1           8h 
playlists-db    1/1     1            1           8h 
videos-api      1/1     1            1           8h 
videos-db       1/1     1            1           8h 
videos-web      1/1     1            1           8h 

kubectl get deploy playlists-api -o yaml | linkerd inject - | kubectl apply -f -
kubectl get deploy playlists-db -o yaml | linkerd inject - | kubectl apply -f -
kubectl get deploy videos-api -o yaml | linkerd inject - | kubectl apply -f -
kubectl get deploy videos-db -o yaml | linkerd inject - | kubectl apply -f -
kubectl get deploy videos-web -o yaml | linkerd inject - | kubectl apply -f -
kubectl -n ingress-nginx get deploy nginx-ingress-controller  -o yaml | linkerd inject - | kubectl apply -f -

```

# Generate some traffic

Let's run a `curl` loop to generate some traffic to our site </br>
We'll make a call to `/home/` and to simulate the browser making a call to get the playlists, <br/>
we'll make a follow up call to `/api/playlists`

# Sample Powershell command. 
Run in Powershell

```
While ($true) { curl -UseBasicParsing http://servicemesh.demo/home/;curl -UseBasicParsing http://servicemesh.demo/api/playlists; Start-Sleep -Seconds 18;}

linkerd -n default check --proxy

linkerd -n default stat deploy

```

# Add Faulty behaviour in videos API
# To simulate failures and test Retries. Set the env variable FLAKY to true, this will make some request fail. then retries profile will help to keep service at 100% request handle.

```
kubectl edit deploy videos-api

#set environment FLAKY=true
```

# Service Profile 
Testing retries

```
linkerd viz profile -n default videos-api --tap deploy/videos-api --tap-duration 10s
```

After crafting the `serviceprofile`, we can apply it using `kubectl`

```
 kubectl apply -f kubernetes/servicemesh/linkerd/serviceprofiles/videos-api-retries.yaml
```

We can see that service profile helps us add retry policies in place: <br/>

```
linkerd routes -n default deploy/playlists-api --to svc/videos-api -o wide
linkerd top deploy/videos-api
```

# Mutual TLS

We can validate if mTLS is working 

```
/work # linkerd -n default edges deployment
SRC                  DST             SRC_NS    DST_NS    SECURED       
playlists-api        videos-api      default   default   √
linkerd-prometheus   playlists-api   linkerd   default   √
linkerd-prometheus   playlists-db    linkerd   default   √
linkerd-prometheus   videos-api      linkerd   default   √
linkerd-prometheus   videos-db       linkerd   default   √
linkerd-prometheus   videos-web      linkerd   default   √
linkerd-tap          playlists-api   linkerd   default   √
linkerd-tap          playlists-db    linkerd   default   √
linkerd-tap          videos-api      linkerd   default   √
linkerd-tap          videos-db       linkerd   default   √
linkerd-tap          videos-web      linkerd   default   √

linkerd -n default tap deploy

```

# ================== Deployment control =================
# Create canary versions


Build new image
# in this case alfavi is my dockerID

```
docker build -t videos-api:canary .
```

```
docker tag videos-api:canary alfavi/videos-api:canary
```

```
docker push alfavi/videos-api:canary
```

```
docker.io/alfavi/videos-api:canary
```


Deploy new testing codes
```
kubectl apply -f kubernetes/servicemesh/applications/videos-api-canary/
```

```
kubectl apply -f kubernetes/servicemesh/applications/videos-db-canary/
```

```
kubectl apply -f kubernetes/servicemesh/applications/videos-web-canary
```

--> Mesh new deployments

```
kubectl get deploy videos-api-canary -o yaml | linkerd inject - | kubectl apply -f -
```

```
kubectl get deploy videos-db-canary -o yaml | linkerd inject - | kubectl apply -f -
```

```
kubectl get deploy videos-web-canary -o yaml | linkerd inject - | kubectl apply -f -
```


--> Split traffic sample

```
kubectl apply -f kubernetes/servicemesh/applications/splitweb
```

```
kubectl apply -f kubernetes/servicemesh/applications/splitvideo
```

-->Check split traffic

```
kubectl -n default get trafficsplit -o yaml
```

Check routes

```
linkerd routes -n default deploy/playlists-api --to svc/videos-api -o wide
```

```
linkerd top deploy/videos-api
```
