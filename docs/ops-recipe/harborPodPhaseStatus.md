---

title: "harborPodPhaseStatus"

---

This alert fires because one or more Harbor pods are in a failing, pending or unknown [phase](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/). This could occur during deployment or at any point during Harbors lifecycle. 


**Harbor exporter**

This is a specific case for when a disruption occurs with the `harbor-exporter` pod, which might lead to issues with exporting metrics to Prometheus, and connecting with Postgres.

Likely you wont see this in the terminal and instead will receive the notification through prometheus. Restart the harbor-exporter pod and check the logs.

```

kubectl -n harbor-cluster delete pod harbor-exporter

kubectl -n harbor-cluster logs harbor-exporter

```

The error is likely to do with the connection to the exporter and other components being down. Try restarting the the deployment. 

```

kubectl -n harbor-cluster delete deployment harbor-exporter

```

If this doesn't work and you have explored other options a likely fix is to re-deploy Harbor itself. As long as you have backups correctly configured to pull from your s3 bucket this should mean that images are correctly restored. Other data such as users assigned to projects will likely be lost and need to be reconfigured. 

**General**

In general if your pods are in a failed, pending or unknown phase you should double check your configurations and the state you expect the pods to be in.

Check what the logs for the pod says:

```

kubectl -n harbor-cluster logs harbor-$COMPONENT

```

You can also describe the pod:

```

kubectl -n harbor-cluster describe pod harbor-$COMPONENT

```

Based on what the logs say try to figure out what is causing the error. Usually it will be the result of an improper configuration. Most configuration for the cluster is created in the [fullstack.yaml](https://github.com/goharbor/harbor-operator/blob/master/manifests/samples/full_stack.yaml). Below are some things to look out for if you cannot find the cause.

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

- Ensure that the `harbor-operator-app` is created in the `harbor-operator` namespace. This is because helm relies on this for creation of some resources.

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
    name: letsencrypt-staging
    kind: ClusterIssuer
```

- For exampel the above configuration should create a certificate called `sample-public-certificate`.

**Other**

- Check that the URL and DNS for core and notary use the domain for the cluster you're working on in the fullstack.yaml.

- You can find this by using the following command:

```

kubectl get ingress -A

```
- This will return all current ingresses for the cluster and the domain that they use. This domain will be the one you want to append to `core.harbor` and `notary.harbor`. For example Grizzly cluster would be: `core.harbor.happaapi.grizzly.gaws.gigantic.io`.


- Check that the nginx ingress class has the following annotation:
  `ingressclass.kubernetes.io/is-default-class: "true"`

- Clean up any crds or resources left over from a previous deployment. This should be done automatically but if not you may get an error complaining that resources such as webhooks and endpoints already exist. Or that the current CRD you are trying to deploy doesn't match the one that already exists. 

- Check that your storage options have been configured correctly. Do they have enough memory? Are they pointing to the correct bucket? Do the credentials exist?

