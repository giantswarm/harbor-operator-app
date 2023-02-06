---


title: "Harbor Down/Harbor Unhealthy"

  
---

This alert fires because a component (eg core, registry, portal etc) of Harbor is either down or unhealthy.

When a component of Harbor is down it means that the pods are crashing off or failing to create.

When a component of Harbor is unhealthy it means it is failing healthchecks. This could mean that the component either is not running or is is not ready.

The first step is always to check the appropriate logs.

```

kubectl -n harbor-cluster logs harbor-$COMPONENT

```

Based on what the logs say try to figure out what is causing the error. Usually it will be the result of an improper configuration. Most configuration for the cluster is created in harbors [fullstack.yaml](https://github.com/goharbor/harbor-operator/blob/master/manifests/samples/full_stack.yaml). Below are some things to look out for if you cannot find the cause.

**Harbor crashing off**

Harbor could crash off for a number of reasons related to the current state of your deployment or cluster. In the below example we can see that the memory is full for Postgres, meaning that core cannot properly interact with it. After digging deeper into the issue we found that even though the memory for Postgres was full, it was the nodes that were using too much CPU, which caused the issue.

```

kubectl get pods -n harbor-cluster

  

harbor-cluster-harbor-harbor-chartmuseum-84bc57ff84-ntxtr 1/1 Running 0 11h

harbor-cluster-harbor-harbor-core-576c845687-fvfgk 0/1 CrashLoopBackOff 131 (3m18s ago) 11h

harbor-cluster-harbor-harbor-exporter-7566d88b64-dnbvd 0/1 Running 232 (5m9s ago) 11h

harbor-cluster-harbor-harbor-jobservice-7d5d965497-krp6q 0/1 CrashLoopBackOff 257 (3m22s ago) 5d17h

harbor-cluster-harbor-harbor-notaryserver-5cb6cb5648-njc95 1/1 Running 5 (5d17h ago) 5d17h

harbor-cluster-harbor-harbor-notarysigner-59749cbd86-mvx5q 1/1 Running 5 (5d17h ago) 5d17h

harbor-cluster-harbor-harbor-portal-84867fd78b-ljlj9 1/1 Running 0 5d17h

harbor-cluster-harbor-harbor-registry-6964c98cc9-ggxqn 2/2 Running 0 11h

harbor-cluster-harbor-harbor-trivy-f9c449849-qtvjf 1/1 Running 0 5d17h

harbor-config-operator-controller-manager-85dff7b5bd-dft7q 0/1 CrashLoopBackOff 85 (2m46s ago) 10h

postgresql-harbor-cluster-harbor-cluster-0 0/1 Running 0 5d17h

postgresql-harbor-cluster-harbor-cluster-1 0/1 Running 0 5d12h

rfr-harbor-cluster-redis-0 1/1 Running 0 11h

rfs-harbor-cluster-redis-745b9755dd-rnw4w 1/1 Running 0 2d3h

```

We can check the logs of the core component.

`kubectl logs -n harbor-cluster harbor-cluster-harbor-harbor-core-576c845687-fvfgk`

```

2023-02-01T10:24:10Z [INFO] [/common/dao/base.go:67]: Registering database: type-PostgreSQL host-postgresql-harbor-cluster-harbor-cluster.harbor-cluster.svc port-5432 database-core sslmode-"disable"

2023-02-01T10:24:10Z [INFO] [/lib/metric/server.go:23]: Prometheus metric server running on port 8001

2023-02-01T10:24:11Z [ERROR] [/common/utils/utils.go:108]: failed to connect to tcp://postgresql-harbor-cluster-harbor-cluster.harbor-cluster.svc:5432, retry after 2 seconds :dial tcp 172.31.45.132:5432: i/o timeout

2023-02-01T10:24:15Z [ERROR] [/common/utils/utils.go:108]: failed to connect to tcp://postgresql-harbor-cluster-harbor-cluster.harbor-cluster.svc:5432, retry after 2 seconds :dial tcp 172.31.45.132:5432: i/o timeout

2023-02-01T10:24:21Z [ERROR] [/common/utils/utils.go:108]: failed to connect to tcp://postgresql-harbor-cluster-harbor-cluster.harbor-cluster.svc:5432, retry after 2 seconds :dial tcp 172.31.45.132:5432: i/o timeout

2023-02-01T10:24:31Z [ERROR] [/common/utils/utils.go:108]: failed to connect to tcp://postgresql-harbor-cluster-harbor-cluster.harbor-cluster.svc:5432, retry after 2 seconds :dial tcp 172.31.45.132:5432: i/o timeout

```

By running:

`kubectl logs -n harbor-cluster postgresql-harbor-cluster-harbor-cluster`

We can see that the logs show memory is full:

```

FATAL: could not write lock file "postmaster.pid": No space left on device

```

We can adjust the memory by altering the fullstack-deployment manifest and redeploying it.

```

kind: Zlando/PostgreSQL

spec:

zlandoPostgreSql:

operatorVersion: "1.8.2"

storage: 2Gi <- Update this value to have more space available on the pod.

replicas: 1

image: registry.opensource.zalan.do/acid/spilo-14:2.1-p7

```

After increasing the storage ensure that you have backups configured for the `harbor-operator-user-values` configmap.

```

configAwsOrGcp:

aws_region: us-east-1

wal_s3_bucket: harbor-postgres-operator

backup:

enabled: true

number_of_backups_to_retain: 5

backup_schedule: "30 * * * *"

bucket: "harbor-postgres-operator"

aws:

region: "us-east-1"

restore:

enabled: true

```

Having applied these two configurations we can delete the postgres and core pod in `harbor-cluster` namespace. If this doesn't solve the issue you may want to look into the state of the nodes and their available memory. As a last resort you can do a clean install of harbor with the backups and memory appropriately configured. A clean install with the right parameters, pointing to a valid backup, will restore projects and images. This means having the `restore` flag enabled. Configuration for things like harbor users may be lost.

**Pod not in a running state**

If you're trying to get a pod from a running to a ready state consider changing the thresholds for readiness probe. This isn't a viable long term fix, but could help with debugging. If pods are taking a long time to become ready it could be due to a high volume of objects in the etcd.

```

kubectl -n harbor-cluster edit pod harbor-$COMPONENT

readinessProbe:
  failureThreshold: 3
  httpGet:
    path: /api/v2.0/ping
    port: http
    scheme: HTTP
  periodSeconds: 10
  successThreshold: 1
  timeoutSeconds: 1

```

From here you should increase the faliure threshold to something like 30 or increase the periodSeconds. This will give your pod more attempts to reach a ready state.

During this time you can SSH onto the pod to retrieve more information about what is happening and why your pod may not be coming up.
  

It may be worth checking the kubelet logs on the node to see if there is any information about the healthiness of the pod.


```

kubectl -n harbor-cluster get pods -owide

```

Check what node the pod is associated with and SSH on to it. You can SSH onto the node using these [plugins](https://github.com/luksa/kubectl-plugins).

```
cat /var/logs/syslog | grep harbor

```

**Secrets**

- Check whether there is a a secret in the harbor-cluster namespace containing the AWS authentication for the the s3 bucket used for backups.

- Check if the secret is referenced in your pods definition. This can be configured in your fullstack for launching harbor under: storage -> s3.

- Check that secrets exist for:

- harbor-core-secrets: `harbor-cluster-harbor-harbor-core`, `harbor-cluster-harbor-harbor-core-csrf`, `harbor-cluster-harbor-harbor-core-encryptionkey`, `harbor-cluster-harbor-harbor-core-secret`, `harbor-cluster-harbor-harbor-core-tokencert`.

- harbor-jobservice-secret: `harbor-cluster-harbor-harbor-jobservice-secret`

- harbor-notaryserver-secret: `harbor-cluster-harbor-harbor-notaryserver-authentication`

- harbor-notarysigner-secrets: `harbor-cluster-harbor-harbor-notarysigner-authentication`, `harbor-cluster-harbor-harbor-notarysigner-authority`, `harbor-cluster-harbor-harbor-notarysigner-encryption-key`.

- harbor-registry-secrets: `harbor-cluster-harbor-harbor-registry-basicauth`, `harbor-cluster-harbor-harbor-registry-http`

- harbor-trivy-secret: `harbor-cluster-harbor-harbor-trivy`

- harbor-redis-secret: `harbor-cluster-redis`

- harbor-posthresql-secret: `harbor.postgresql-harbor-cluster-harbor-cluster.credentials`

**Namespaces**

- Ensure that the `harbor-operator-app` is created in the `harbor-operator` namespace. This is because helm relies on this specific namespace name for creation of some resources.

- Equally, ensure that the components are being created in the `harbor-cluster` namespace. This is because it is hardcoded into a rolebinding for PodSecurityPolicies.

- These components will be deployments named: `harbor-cluster-harbor-harbor-chartmuseum`, `harbor-cluster-harbor-harbor-core`, `harbor-cluster-harbor-harbor-notaryserver`, `harbor-cluster-harbor-harbor-notarysigner`, `harbor-cluster-harbor-harbor-portal`, `harbor-cluster-harbor-harbor-registry`, `harbor-cluster-harbor-harbor-trivy`, `rfs-harbor-cluster-redis`.

**Certificates**

- Check that certificates have been properly issued. In the fullstack.yaml the clusterissuer should be set to `letsencrypt-giantswarm`.

- The certificate should have the same name as you have given it in the `fullstack.yaml`.

```

apiVersion: cert-manager.io/v1

kind: Certificate

metadata:

name: sample-public-certificate

namespace: harbor-cluster

spec:

secretName: sample-public-certificate

dnsNames:

- core.harbor.localhost

- notary.harbor.localhost

issuerRef:

name: letsencrypt-giantswarm

kind: ClusterIssuer

```

- For example the above configuration should create a certificate called `sample-public-certificate`.

**Ingress**

- Sometimes the portal will 404 and the website will be inaccessible. This might be because the ingress class for harbor harbor core is not set.

```

kubectl edit ingress -n harbor-cluster harbor-cluster-harbor-harbor-giantswarm

```

- Check that the ingress class for harbor has: `ingressClassName: nginx` added underneath spec. If not add it like so:

 spec:
    ingressClassName: nginx

- Deleting the harbor core pod may cause the ingress spec to lose the key value pair. It should be the first thing you check if you are getting 404.



- Check that the URL and DNS for core and notary use the domain for the cluster you're working on in the fullstack.yaml.

- You can find this by using the following command:

```

kubectl get ingress -A

```

- This will return all current ingresses for the cluster and the domain that they use. This domain will be the one you want to append to `core.harbor` and `notary.harbor`. For example Grizzly cluster would be: `core.harbor.happaapi.grizzly.gaws.gigantic.io`.

- Check that the nginx ingress class has the following annotation:

`ingressclass.kubernetes.io/is-default-class: "true"`

**Other**

- Clean up any crds or resources left over from a previous deployment. This should be done automatically but if not you may get an error complaining that resources such as webhooks and endpoints already exist. Or that the current CRD you are trying to deploy doesn't match the one that already exists.

- Check that your storage options have been configured correctly. Do they have enough memory? Are they pointing to the correct bucket? Do the credentials exist?

For this I have provided an example of correct configuration for immutable fields in the `harbor-operator-user-values` configmap.

```
configKubernetes:

secret_name_template: "{username}.{cluster}.credentials"

enable_pod_antiaffinity: "true"

pod_environment_configmap: "harbor-operator/pod-config"

pod_environment_secret: "pod-secret"
```
