## Prerequisites

- [golang](https://go.dev/doc/install)
- [docker](https://docs.docker.com/engine/install)
- [kubectl](https://kubernetes.io/docs/tasks/tools)
- [minikube](https://minikube.sigs.k8s.io/docs/start/?arch=%2Fmacos%2Farm64%2Fstable%2Fbinary+download)

## Initialize a new project
```
kubebuilder init --domain kb.dev --repo github.com/jiaqi-yin/ghost-controller
```

## Scaffold a Kubernetes API
```
kubebuilder create api \
  --kind Ghost \
  --group marketing \
  --version v1 \
  --resource true \
  --controller true
```

## Scaffold a webhook for an API resource
```
kubebuilder create webhook \
  --kind Ghost \
  --group marketing \
  --version v1 \
  --defaulting \
  --programmatic-validation \
  --conversion
```

## Install dependencies: cert-manager, ingress-nginx and prometheus-operator
```
make deps
```
OR

```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.16.1/cert-manager.yaml
kubectl create -f config/prometheus-operator/bundle.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.11.2/deploy/static/provider/cloud/deploy.yaml
```

## Install CRDs into the K8s cluster
```
make install
```
```
kubectl get crd ghosts.marketing.kb.dev --output jsonpath="{.spec.versions[0].schema['openAPIV3Schema'].properties.spec.properties}{\"\n\"}" | jq

// output:
{
  "enableIngress": {
    "type": "boolean"
  },
  "imageTag": {
    "pattern": "^[-a-z0-9]*$",
    "type": "string"
  },
  "replicas": {
    "format": "int32",
    "maximum": 3,
    "minimum": 1,
    "type": "integer"
  }
}
```

## Deploy
```
export IMG=jiaqiyin/ghost-controller:latest
make docker-build docker-push deploy
```

## Create two sample ghost applications in different namespaces
```
kubectl create namespace marketing
kubectl create namespace sales
kubectl apply -f config/samples/marketing_v1_ghost.yaml
```
```
kubectl get deploy,svc,pod,pvc,ingress,ghost -n marketing

// Output
NAME                                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ghost-deployment-marketing   2/2     2            2           14m

NAME                              TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/ghost-service-marketing   NodePort   10.109.209.74   <none>        80:31901/TCP   14m

NAME                                              READY   STATUS    RESTARTS   AGE
pod/ghost-deployment-marketing-688c5d95df-bnvsw   1/1     Running   0          6s
pod/ghost-deployment-marketing-688c5d95df-hmgjz   1/1     Running   0          10s

NAME                                             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/ghost-data-pvc-marketing   Bound    pvc-6c1299a5-5691-491c-9af1-a810863e371c   1Gi        RWO            standard       <unset>                 14m

NAME                                                CLASS   HOSTS                  ADDRESS     PORTS   AGE
ingress.networking.k8s.io/ghost-ingress-marketing   nginx   ghost-sample1.kb.dev   127.0.0.1   80      14m

NAME                                   ENABLEINGRESS   REPLICAS   IMAGETAG
ghost.marketing.kb.dev/ghost-sample1   true            2          latest
```
```
kubectl get deploy,svc,pod,pvc,ingress,ghost -n sales

// Output
NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ghost-deployment-sales   1/1     1            1           15m

NAME                          TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/ghost-service-sales   NodePort   10.109.55.200   <none>        80:31803/TCP   15m

NAME                                          READY   STATUS    RESTARTS   AGE
pod/ghost-deployment-sales-649f5cdb95-ltdpf   1/1     Running   0          63s

NAME                                         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/ghost-data-pvc-sales   Bound    pvc-37cd654f-dfdf-4bba-9459-f9f937336361   1Gi        RWO            standard       <unset>                 15m

NAME                                   ENABLEINGRESS   REPLICAS   IMAGETAG
ghost.marketing.kb.dev/ghost-sample2   false           1          alpine

```
## Access the application

*Make sure enableIngress is set to true in the [sample ghost yaml](../config/samples/marketing_v1_ghost.yaml) file so that the applicaiton can be accessible via domain names.*
```
minikube tunnel

curl -IL -v --resolve ghost-sample1.kb.dev:80:127.0.0.1 http://ghost-sample1.kb.dev

curl -IL -v --resolve ghost-sample2.kb.dev:80:127.0.0.1 http://ghost-sample2.kb.dev
```