# Kube Journey Resource Graph Definitions with kro

## Overview

**kro** (Kube Resource Orchestrator) is an open-source, Kubernetes-native project that allows you to define custom Kubernetes APIs using simple and straightforward configuration. With kro, you can easily configure new custom APIs that create a group of Kubernetes objects and the logical operations between them. kro leverages CEL (Common Expression Language), the same language used by Kubernetes webhooks, for logical operations. Using CEL expressions, you can easily pass values from one object to another and incorporate conditionals into your custom API definitions. Based on the CEL expressions, kro automatically calculates the order in which objects should be created. You can define default values for fields in the API specification, streamlining the process for end users who can then effortlessly invoke these custom APIs to create grouped resources.



Visit the kro homepage for the latest information, install, upgrades and install. Below you will find quick install steps to get kro up and running quickly.

[https://kro.run](https://kro.run)

## Quick Install

To Install kro you will need the following:

1. Helm 3.x installed
2. kubectl installed and configured to interact with your Kubernetes cluster

First get the latest version from github and set the **KRO_VERSION** variable

```
export KRO_VERSION=$(curl -sL \
    https://api.github.com/repos/kro-run/kro/releases/latest | \
    jq -r '.tag_name | ltrimstr("v")'
  )
```

Verify that the kro version is set
```
echo $KRO_VERSION
```

Use helm to install the latest version of kro 

```
helm install kro oci://ghcr.io/kro-run/kro/kro \
  --namespace kro \
  --create-namespace \
  --version=${KRO_VERSION}
```

Verify that you have a kro pod running

```
kubectl get pods -n kro
```