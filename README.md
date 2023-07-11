# k8s-jw-scaler

This repository contains Kubernetes manifests to deploy:

  - S3 compatible storage (minimal) - [minio](https://min.io/)
  - [jw-scaler](https://github.com/muawiakh/go-jw-scaler)
  - [NGINX Ingress Controller](https://docs.nginx.com/nginx-ingress-controller/)
    - Using: https://kind.sigs.k8s.io/docs/user/ingress/

We might need to deploy an ingress, evolve into a helm chart or use kustomize based on the time constraints(~2 hours)

## Task workflow

- Deploy Kubernetes clusters with local registry. [tf-jw-scaler](https://github.com/muawiakh/tf-jw-scaler)
- Build and push jw-scaler to local docker registry. [go-jw-scaler](https://github.com/muawiakh/go-jw-scaler)
- Deploy k8s resources: nginx controller, minio, jw-scaler. [k8s-jw-scaler](https://github.com/muawiakh/k8s-jw-scaler)


## Clone the project

```
$ git clone git@github.com:muawiakh/k8s-jw-scaler.git
$ cd k8s-jw-scaler
```

### Pre-requisites

 - [kubectl](https://kubernetes.io/docs/tasks/tools/)
   - This was developed with:
   ```bash
   Client Version: version.Info{Major:"1", Minor:"24", GitVersion:"v1.24.3", GitCommit:"aef86a93758dc3cb2c658dd9657ab4ad4afc21cb", GitTreeState:"clean", BuildDate:"2022-07-13T14:30:46Z", GoVersion:"go1.18.3", Compiler:"gc", Platform:"darwin/amd64"}
   ```
- **kind** clusters deployed using [tf-jw-scaler](https://github.com/muawiakh/tf-jw-scaler)

## Deploy nginx ingress controller

After you have your kind cluster deployed, using [tf-jw-scaler](https://github.com/muawiakh/tf-jw-scalert). 

```bash
$ kubectl config get-contexts
CURRENT   NAME                     CLUSTER                  AUTHINFO                 NAMESPACE
          kind-app-k8s-cluster     kind-app-k8s-cluster     kind-app-k8s-cluster     
*         kind-infra-k8s-cluster   kind-infra-k8s-cluster   kind-infra-k8s-cluster

# Here it is using kind-infra-k8s-cluster by default but we want to deploy it on both
# Deploying for infra cluster
$ kubectl --context kind-infra-k8s-cluster apply -f nginx-ingress/nginx-ingress-controller-all.yaml

# Wait for it to be ready
$ kubectl --context kind-infra-k8s-cluster wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s

# For webapp cluster
$ kubectl --context kind-app-k8s-cluster apply -f nginx-ingress/nginx-ingress-controller-all.yaml

# Wait for it to be ready
$ kubectl --context kind-app-k8s-cluster wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

## Deploy minio

This is only needed for the `infra` cluster.

```bash
$ kubectl --context kind-infra-k8s-cluster apply -f minio/minio.v1.yaml

# wait for deployment
$ kubectl --context kind-infra-k8s-cluster wait --namespace minio-dev \
   --for=condition=ready pod \
   --selector=app=minio \
   --timeout=90s
```

## Deploy jw-scaler
This should only be deployed on the webapp cluster.

```bash
$ kubectl --context kind-app-k8s-cluster apply -f jw-scaler/jw-scaler.v1.yaml

# wait for deployment
$ kubectl --context kind-app-k8s-cluster wait --namespace jw-scaler \
   --for=condition=ready pod \
   --selector=app=jw-scaler \
   --timeout=90s
```

## Verify deployments

### Minio

```bash
$ kubectl --context kind-infra-k8s-cluster port-forward pod/minio 9000 9090 -n minio-dev
# Check in your browser and login with minioadmin/minioadmin

$ curl localhost:8080/minio/health/live
```

### jw-scaler

```bash
$ curl -v localhost:8080/jw-scaler

# Browser

$ http://localhost:8081/jw-scaler/<s3-bucket>/posters/<movie-name>.jpeg
```