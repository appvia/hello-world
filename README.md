# hello-world

A simple hello-world web application running with a non-root user on Alpine Linux in a Docker container. 

## Prerequisites

Docker
Helm

## Build

```
docker build --platform=linux/amd64 -t hello-world .
```

## Run

```
docker run -it -p 8080:8080 --rm --name hello-world ghcr.io/appvia/hello-world/hello-world:main
```

Visit http://localhost:8080 in a web browser.

## Push to GHCR

### Create a new GitHub Personal Access Token

Follow instructions as seen on:

https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry

# Its not recommended to use your personal ghcr.io repo for this tutorial, if you choose to use one, you will have to create and reference an `ImagePullSecret`
Once you have A PAT token created, you can build an auth string using the following format:

`username:123123adsfasdf123123` where username is your GitHub `username` and `123123adsfasdf123123` is the token with read:packages scope.

let's Base64 encode it first:

`echo -n "username:123123adsfasdf123123" | base64`

Now, paste the encoded string within this json template and base64 it again

`echo -n  '{"auths":{"ghcr.io":{"auth":"<enter base64 encoded string here>"}}}' | base64`

and store it at as `dockerconfigjson.yaml` file.

```
kind: Secret
type: kubernetes.io/dockerconfigjson
apiVersion: v1
metadata:
  name: dockerconfigjson
  labels:
    app: hello-world
data:
  .dockerconfigjson: <enter double base64 encoded string here >
```

And now we have prepared all the pieces, time to create a new secret
`kubectl create -f dockerconfigjson.yaml -n ${KUBE_NAMESPACE}`

### Push image to GHCR

```
REGISTRY=ghcr.io/appvia
GH_REPO=hello-world
IMAGE_NAME=hello-world
TAG=main
IMAGE_URL=${REGISTRY}/${GH_REPO}/${IMAGE_NAME}:${TAG}

docker build --platform=linux/amd64 -t ${IMAGE_URL} .
docker push ${IMAGE_URL}
```

## Deploy to Kubernetes

### Helm Install

```
INGRESS_HOSTNAME=
KUBE_NAMESPACE=hello-world

helm -n ${KUBE_NAMESPACE} install hello-world ./charts/hello-world --set ingress.hostname=${INGRESS_HOSTNAME}
```

### Helm Flux Operator

```
cat <<EOF | kubectl -n ${KUBE_NAMESPACE} apply -f -
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: GitRepository
metadata:
  name: hello-world
spec:
  interval: 1h
  url: https://github.com/appvia/hello-world
  ref:
    branch: main
EOF

cat <<EOF | kubectl -n ${KUBE_NAMESPACE} apply -f -
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: hello-world
spec:
  releaseName: hello-world
  interval: 1h
  chart:
    spec:
      chart: charts/hello-world
      sourceRef:
        kind: GitRepository
        name: hello-world
        namespace: ${KUBE_NAMESPACE}
  values:
    ingress:
      hostname: ${INGRESS_HOSTNAME}
EOF
```

