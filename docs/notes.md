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

## Install CRDs into the K8s cluster
```
make install
```
```
kubectl get crd ghosts.marketing.kb.dev --output jsonpath="{.spec.versions[0].schema['openAPIV3Schema'].properties.spec.properties}{\"\n\"}" | jq

// output:
{
  "imageTag": {
    "pattern": "^[-a-z0-9]*$",
    "type": "string"
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
kubectl create namespace marketing
kubectl apply -f config/samples/marketing_v1_ghost.yaml
```

## View the ghost application in a browser
```
kubectl port-forward service/ghost-service-marketing 8080:80 -n marketing

http://localhost:8080
```
