---

title: "harborPodPhaseStatus"

---

This alert fires because one or more Harbor pods are in a failing, pending or unknown [phase](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/). This could occur during deployment or at any point during Harbors lifecycle. 


**Harbor exporter**

This is a specific case for when a disruption occurs with the `harbor-exporter` pod, which might lead to issues with exporting metrics to Prometheus, and connecting with Postgres.

Likely you wont see this in the terminal and instead will receive the notification through prometheus. Restart the harbor-exporter pod and check the logs.

```

> kubectl -n harbor-cluster delete pod harbor-exporter

> kubectl -n harbor-cluster logs harbor-exporter

```

The error is likely to do with the connection to the exporter and other components being down. Try restarting the service. 

```

> kubectl -n harbor-cluster delete service harbor-exporter

```

If this doesn't work and you have explored other options a likely fix is to re-deploy harbor itself.

**General**

In general if your pods are in a failed, pending or unknown phase you should double check your configurations and the state you expect the pods to be in.

Check what the logs for the pod says:

```

> kubectl -n harbor-cluster logs harbor-$COMPONENT

```

You can also describe the pod:

  

```

> kubectl -n harbor-cluster describe pod harbor-$COMPONENT

```

Based on what the logs say try to figure out what is causing the error. Usually it will be the result of an improper configuration. Most configuration for the cluster is created in the [fullstack.yaml](https://github.com/goharbor/harbor-operator/blob/master/manifests/samples/full_stack.yaml). Below are some things to look out for if you cannot find the cause.


**Secrets**

```

- Check whether there is a a secret in the harbor-cluster namespace containing the AWS authentication for the the s3 bucket used for backups.

- Check if the secret is referenced in your pods definition. This can be configured in your fullstack for launching harbor under: storage -> s3.

- Check that the postgres-pod secret exists with the configurations for postgres correctly enabled. This exists as `pod-secret`.

- Check that secrets exist for: redis, postgres, core, jobservice, notaryserver, notarysigner, registry and trivy.

```

**Namespaces**

```

- Ensure that the `harbor-operator-app` is created in the `harbor-operator` namespace. This is because helm relies on this for creation of some resources.

- Equally, ensure that the components are being created in the `harbor-cluster` namespace. This is because it is hardcoded into a rolebinding for PodSecurityPolicies.

```

**Certificates**

```

- Check that certificates have been properly issued. In the fullstack.yaml the clusterissuer should be set to `letsencrypt-giantswarm`.

```

**Other**

```

- Check that the URL and DNS for core and notary are set correctly for the cluster you're working on in the fullstack.yaml.

- Check that the nginx ingress class has the following annotation:

`ingressclass.kubernetes.io/is-default-class: "true"`

- Clean up any crds or resources left over from a previous deployment.

- Check that your storage options have been configured correctly. Do they have enough memory? Are they pointing to the correct bucket? Do the credentials exist?

```
