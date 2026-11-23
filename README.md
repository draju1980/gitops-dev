# gitops-sandbox
## Overview

A GitOps sandbox environment for testing Kubernetes deployments.

## Prerequisites

Before executing any Helmfile commands, ensure your active Kubernetes context is set to `gitops-dev`:

```sh
kubectx gitops-dev
kubectl config current-context
```

## Deployment

Deploy all cluster services from the root of the gitops-sandbox repository:

```sh
helmfile template . | kubectl create -f - 2>/dev/null
helmfile template . | kubectl apply -f - --server-side=true 2>/dev/null
```


### Testing Cluster Services

To test cluster services locally, use the relative path:

```sh
- path: ../do-kubernetes-cluster-module/cluster-services/helmfile.yaml.gotmpl
```

To test cluster services using a specific GitHub tag, use the git reference:

```sh
- path: git::https://git:{{ .Values.github.accesstoken }}@github.com/blessnetwork/do-kubernetes-cluster-module.git@cluster-services/helmfile.yaml.gotmpl?ref=1.0.21
```
