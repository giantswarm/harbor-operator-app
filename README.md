# harbor-operator-app chart

[![CircleCI](https://circleci.com/gh/giantswarm/harbor-operator-app.svg?style=shield)](https://circleci.com/gh/giantswarm/harbor-operator-app)

Giant Swarm offers a harbor-operator-app App which can be installed in workload clusters.
Here we define the harbor-operator-app chart with its templates and default configuration.

## Installing

There are several ways to install this app onto a workload cluster.

- [Using our web interface](https://docs.giantswarm.io/ui-api/web/app-platform/#installing-an-app).
- By creating an [App resource](https://docs.giantswarm.io/ui-api/management-api/crd/apps.application.giantswarm.io/) in the management cluster as explained in [Getting started with App Platform](https://docs.giantswarm.io/app-platform/getting-started/).

## Configuring

### values.yaml

**This is an example of a values file you could upload using our web interface.**

Define a `configmap` which specifies the values required.

```yaml
installCRDs: true
redis-operator:
  enabled: true
minio-operator:
  enabled: false
postgres-operator:
  enabled: true
  image:
    tag: v1.8.2
  configGeneral:
    docker_image: registry.opensource.zalan.do/acid/spilo-14:2.1-p7
  configKubernetes:
    secret_name_template: "{username}.{cluster}.credentials"
    enable_pod_antiaffinity: "true"
    pod_environment_configmap: "harbor-operator-ns/pod-config"
  configAwsOrGcp:
    aws_region: <bucket-region>
    wal_s3_bucket: <s3-bucket-name>

```

### Sample App CR and ConfigMap for the management cluster

If you have access to the Kubernetes API on the management cluster, you could create
the App CR and ConfigMap directly.

Here is an example that would install the app to
workload cluster `abc12`:

```yaml
# appCR.yaml
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
      name: "harbor-operator-user-values" # This should be the same as the name of the configmap created in step 1
      namespace: "harbor-operator-ns"
  version: <harbor-operator-version>

```

```yaml
# user-values-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: harbor-operator-user-values
  namespace: harbor-operator-ns
data:
  values: |
    installCRDs: true
    redis-operator:
      enabled: true
    minio-operator:
      enabled: false
    postgres-operator:
      enabled: true
      image:
        tag: v1.8.2
      configGeneral:
        docker_image: registry.opensource.zalan.do/acid/spilo-14:2.1-p7
      configKubernetes:
        secret_name_template: "{username}.{cluster}.credentials"
        enable_pod_antiaffinity: "true"
        pod_environment_configmap: "harbor-operator-ns/pod-config"
      configAwsOrGcp:
        aws_region: <bucket-region>
        wal_s3_bucket: <s3-bucket-name>

```

See our [full reference on how to configure apps](https://docs.giantswarm.io/app-platform/app-configuration/) for more details.

## Compatibility

This app has been tested to work with the following workload cluster release versions:

- _add release version_

## Limitations

Some apps have restrictions on how they can be deployed.
Not following these limitations will most likely result in a broken deployment.

- _add limitation_

## Credit

- [harbor-operator](https://github.com/goharbor/harbor-operator/blob/master/charts/harbor-operator/Chart.yaml)
