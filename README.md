# Policy-based Resource Optimization

This Nirmata solution uses [Kyverno polices](https://kyverno.io/) and the [Kubernetes VPA recommender](https://github.com/kubernetes/autoscaler/blob/master/vertical-pod-autoscaler/pkg/recommender/README.md) to help optimize resource allocations for workloads. 

## Description

A [Kyverno generate policy](./config/kyverno/policies/generate-vpa.yaml) is used to generate a VerticalPodAutocaler (VPA) for Deployment and StatefulSet resources, with the VPA updater mode set to `Off`. This allows the VPA Recommender to observe resource metrics and provide recommendations for each container, without making any changes.

Next, a [Kyverno validate policy](./config/kyverno/policies/generate-vpa.yaml) is used to periodically each resource and compare its resource requests to the upper and lower bounds reported by the VPA recommender. 

The policy rule allows for a (customizable) 20% deviation i.e. only reports violations if the request is 20% higher than the upperbound, or 20% lower than the lowerbound. The rule also handles workloads where no resource request is set, and recommends a setting.

For demo purposes, the rule processes VPA recommendations 1 minute after creation. For production, this should be updated to a higher value.

## Benefits

Leveraging Kyverno's native policies, policy reports, exceptions, and other features provides an easy to use, scalable, and customizable solution for resource optimization and tuning. Developers can easily access and review violations using native tools, and no additional controllers are required.

## Demo

This demo script uses a `kind` cluster and `helm`. 

If you do not have `kind` installed use:  

   https://kind.sigs.k8s.io/docs/user/quick-start#installation

To install `helm` use:

   https://helm.sh/docs/helm/helm_install/

### Install the Kubernetes Metrics Server

The metrics service is required. You can check if one is installed using:

```sh
kubectl top pods -A
```

To install the metrics server, execute:

```sh
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm repo update
helm upgrade --install --set args={--kubelet-insecure-tls} metrics-server metrics-server/metrics-server --namespace kube-system
```

### Install Prometheus (optional)

Prometheus is optional, but can help the VPA recommender with historical data for accuracy. 

To install the Prometheus, execute:

```sh
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/prometheus -n prometheus --create-namespace
```

NOTE: if you chose not to install Prometheus, remove or update the following lines in the VPA recommender installation:

```yaml
- --storage=prometheus
- --prometheus-address=http://prometheus-server.default.svc.cluster.local:80
```

### Install VPA

The [Kubernetes VerticalPodAutoscaler](https://kubernetes.io/docs/concepts/workloads/autoscaling/) has several components:
* Recommender
* Updater
* Admission Plugin

For this solution, we only need the `VPA Recommender`. You can install it by executing:

```sh
kubectl apply -f https://raw.githubusercontent.com/nirmata/resource-optimizer/main/config/vpa/install-vpa-recommender.yaml
```

Alternatively, you can use this [helm chart](https://artifacthub.io/packages/helm/fairwinds-stable/vpa) to install, but make sure you customize the arguments correctly.

See [Components of VPA](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler#components-of-vpa) for details on all components.


### Install Kyverno

Execute the following commands to install Kyverno:

```sh
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update
helm install kyverno kyverno/kyverno -n kyverno --create-namespace
```

### Configure Kyverno RBAC permissions

To generate a VerticalPodAutoscale, Kyverno needs to be given additional permissions for VPA resources.

Execute the following command to configure Kyverno permissions:

```sh
kubectl apply -f https://raw.githubusercontent.com/nirmata/resource-optimizer/main/config/kyverno/rbac.yaml
```

### Install Kyverno Policies

Next, install Kyverno policies:

```sh
kubectl apply -f https://raw.githubusercontent.com/nirmata/resource-optimizer/main/config/kyverno/policies/generate-vpa.yaml
kubectl apply -f https://raw.githubusercontent.com/nirmata/resource-optimizer/main/config/kyverno/policies/check-resources.yaml
```

The policies are configured to 

## Run a workload

Install a workload:

```sh
kubectl apply -f https://raw.githubusercontent.com/nirmata/resource-optimizer/main/config/workload/demo-kyverno-vpa.yaml
```

The VPA will start providing recommendations after around 60s:

```sh
kubectl -n demo-kyverno-vpa get vpa -w
```

```sh
NAME           MODE   CPU   MEM   PROVIDED   AGE
demo-kyverno   Off                False      38s
demo-kyverno   Off    15m   104857600   True       74s
```

You can check the namespace for policy violations to be reported (may take a few minutes):

```sh
kubectl -n demo-kyverno-vpa get polr
```

```sh
NAME                                   KIND         NAME   PASS   FAIL   WARN   ERROR   SKIP   AGE
fb7b3a29-3b37-44d6-b1a5-c2eee2ca41c8   Deployment   demo   1      1      0      0       0      3s
```

```sh
kubectl -n demo-kyverno-vpa get polr -o yaml | grep "result: fail" -A 10 -B 3
```

```sh
  - message: 'validation failure: underprovisioned resources: set memory.request (50Mi)
      > 104857600'
    policy: check-resources
    result: fail
    rule: memory
    scored: true
    source: kyverno
    timestamp:
      nanos: 0
      seconds: 1721083772
  scope:
    apiVersion: apps/v1
    kind: Deployment
    name: demo
```

Try changing the CPU and memory requests, and apply policies:

```sh
kubectl -n demo-kyverno-vpa set resources deploy demo --limits memory=500Mi --requests memory=500Mi
```

Using the [Kyverno CLI](https://kyverno.io/docs/kyverno-cli/install/):

```sh
kubectl-kyverno apply config/kyverno/policies/check-resources.yaml --cluster  -p -n demo-kyverno-vpa
```

This should show both rules as passing:

```yaml
# <snipped>
summary:
  error: 0
  fail: 0
  pass: 2
  skip: 0
  warn: 0
```

You can check the VPA recomnmendations in the `status.recommendation.containerRecommendations` field:

```sh
kubectl -n demo-kyverno-vpa get vpa -o yaml
```

```yaml
  status:
    conditions:
    - lastTransitionTime: "2024-07-15T23:08:22Z"
      status: "True"
      type: RecommendationProvided
    recommendation:
      containerRecommendations:
      - containerName: demo
        lowerBound:
          cpu: 15m
          memory: "104857600"
        target:
          cpu: 15m
          memory: "104857600"
        uncappedTarget:
          cpu: 15m
          memory: "104857600"
        upperBound:
          cpu: 243m
          memory: "1099208610"
```

The VPA Recommender will improve the upper and lower bound recommendations over time.

For demo purposes [policy rule](./config/kyverno/policies/check-resources.yaml) only waits for `0` minutes to start using VPA recommendations.

For production, this should be set to a much higher amount e.g `24h`.


## (Advanced) Apply load to the workload

The sample workloads is configured with an HPA. You can apply load to the workload, and scale it to 3 instances:

```sh
kubectl run -i --tty load-generator --rm --image=busybox --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://demo.demo-kyverno-vpa.svc.cluster.local:80; done"
```

The VPA will record the upper and lower bounds based on the load.



