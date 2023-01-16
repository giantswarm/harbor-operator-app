# Sample App CR and ConfigMap for the management cluster
If you have access to the Kubernetes API on the management cluster, you could create the App CR and ConfigMap directly.

## Prerequisites

* cert-manager
* Ingress controller
* If you are using nginx ingress controller make sure you have annotated your ingressClass:
  `ingressclass.kubernetes.io/is-default-class: "true"`
<hr />

Here is an example that would install the app to management cluster ginger:

```yaml
apiVersion: v1
data:
  values: |
    installCRDs: true
    redis-operator:
      enabled: true
    minio-operator:
      enabled: false
    postgres-operator:
      enabled: true
kind: ConfigMap
metadata:
  name: harbor-operator-user-values
  namespace: harbor-operator
---
apiVersion: application.giantswarm.io/v1alpha1
kind: App
metadata:
  labels:
    app-operator.giantswarm.io/version: 0.0.0
    app.kubernetes.io/name: harbor-operator
  name: harbor-operator
  namespace: giantswarm   
spec:
  catalog: giantswarm-playground-test
  kubeConfig:
    inCluster: true
  name: harbor-operator
  namespace: harbor-operator-ns
  userConfig:
    configMap:
      name: "harbor-operator-user-values"
      namespace: "harbor-operator"
  version: 0.0.0-a1333bd7599ddfa8f5c7fb6ce26dc777b84fa7b3
  ```
