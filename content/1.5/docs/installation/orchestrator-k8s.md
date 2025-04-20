---
title: "Orchestrator on Kubernetes"
date: 2024-04-09
---

The following guide is for installing on a Kubernetes cluster. It is well tested and working in CI with a [kind](https://kind.sigs.k8s.io/) installation.

Here's a kind configuration that is easy to work with (the apiserver port is static, so the kubeconfig is always the same)
```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  apiServerAddress: "127.0.0.1"
  apiServerPort: 16443
nodes:
  - role: control-plane
    kubeadmConfigPatches:
    - |
      kind: InitConfiguration
      nodeRegistration:
        kubeletExtraArgs:
          node-labels: "ingress-ready=true"
    - |
      kind: KubeletConfiguration
      localStorageCapacityIsolation: true
    extraPortMappings:
      - containerPort: 80
        hostPort: 9090
        protocol: TCP
      - containerPort: 443
        hostPort: 9443
        protocol: TCP
  - role: worker
```

Save this file as `kind-config.yaml`, and now run:
```bash
kind create cluster --config kind-config.yaml
kubectl apply -f https://projectcontour.io/quickstart/contour.yaml
kubectl patch daemonsets -n projectcontour envoy -p '{"spec":{"template":{"spec":{"nodeSelector":{"ingress-ready":"true"},"tolerations":[{"key":"node-role.kubernetes.io/control-plane","operator":"Equal","effect":"NoSchedule"},{"key":"node-role.kubernetes.io/master","operator":"Equal","effect":"NoSchedule"}]}}}}'
```

The cluster should be up and running with [Contour ingress-controller](https://projectcontour.io) installed, so localhost:9090 will direct the traffic to Backstage, because of the ingress created by the helm chart on port 80.

{{< remoteMD "https://raw.githubusercontent.com/rhdhorchestrator/orchestrator-helm-chart/main/charts/orchestrator-k8s/README.md" >}}
