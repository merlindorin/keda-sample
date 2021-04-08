# KEDA example

This repository consists of everything you need to setup simple Kubernetes 
cluster and demonstrate usage of KEDA redis and cron scalers. For more
samples check https://github.com/kedacore/samples

The included `helper` provides an easy way to perform both 0 -> n and n -> 0 scalings.  

## Create cluster

The deployment consists of 3 components:
- Redis instance
- Dummy pod that will be scaled up and down
- App service that provides some helper methods

```sh
./scripts/build.sh

kind create cluster

./scripts/kind-load.sh

kubectl apply -f deployment/
```

## Install KEDA

Deploying KEDA with Helm is very simple:

- Add Helm repo: `helm repo add kedacore https://kedacore.github.io/charts`
- Update Helm repo: `helm repo update`
- Install keda Helm chart

```shell script
kubectl create namespace keda
helm install keda kedacore/keda --namespace keda
```

## Observe
To observe how everything works you can watch two things:

- number of pods and their state: `watch -n2 "kubectl get pods"`
- HPA stats: `watch -n2 "kubectl get hpa"`

## Cron example
To scale the server deployment using [Cron scaler](https://keda.sh/scalers/cron/) first we have 
to deploy the `ScaledObjects`:

```sh
kubectl apply -f keda/cron-hpa.yaml
```

this should result in creation of a new `ScaledObjects` and new HPA.

Scale will be performed down on odd minutes add restored to default replica on even minutes.

## Redis example
To scale the dummy deployment using [Redis scaler](https://keda.sh/scalers/redis-lists/) first we have 
to deploy the `ScaledObjects`:

```sh
kubectl apply -f keda/redis-hpa.yaml
```

this should result in creation of a new `ScaledObjects` and new HPA

```sh
# kubectl get scaledobjects
NAME                 DEPLOYMENT   TRIGGERS   AGE
redis-scaledobject   dummy        redis      5s

# kubectl get hpa
NAME             REFERENCE          TARGETS              MINPODS   MAXPODS   REPLICAS   AGE
keda-hpa-dummy   Deployment/dummy   <unknown>/10 (avg)   1         4         0          45s
```

To scale up we have to populate the Redis queue. To do this we can use the helper app:
```shell script
kubectl exec -it $(k get pods | grep "server" | cut -f 1 -d " ") /go/bin/keda-talk redis publish
```
and to scale down:
```shell script
kubectl exec -it $(k get pods | grep "server" | cut -f 1 -d " ") /go/bin/keda-talk redis drain
```