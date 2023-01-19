---

title: "Harbor Unhealthy"

---

This alert fires because a component of Harbor is unhealthy.
  
When a component of harbor if unhealthy it means it is failing healthchecks. This could mean that the component either is not running or is is not ready. Once you have identified which component is unhealthy, you can run the following commands to gain more insight.

```

kubectl -n harbor-cluster logs harbor-$COMPONENT

kubectl -n harbor-cluster describe pod harbor-$COMPONENT

kubectl -n harbor-cluster describe service harbor-$COMPONENT

kubectl -n harbor-operator describe deployment harbor-$COMPONENT

kubectl -n harbor-operator describe statefulset harbor-$COMPONENT


```

If you're trying to get a pod from a running to a ready state consider changing the thresholds for readiness probe. This isn't a viable long term fix, but could help with debugging. If pods are taking a long time to become ready it could be due to a high volume of objects in the etcd.

```

kubectl -n harbor-cluster edit pod harbor-$COMPONENT 

Increase the time out period on the readiness probe to 60 seconds.

```

It may be worth checking the kubelet logs on the node to see if there is any information about the healthiness of the pod.

```

kubectl -n harbor-cluster get pods -owide

```

Check what node the pod is associated with and SSH on to it. You can SSH onto the node using these [plugins](https://github.com/luksa/kubectl-plugins).

```

cat /var/logs/syslog | grep harbor

```

Based on what the logs say try to figure out what is causing the error. Usually it will be the result of an improper configuration. Most configuration for the cluster is created in the [fullstack.yaml](https://github.com/goharbor/harbor-operator/blob/master/manifests/samples/full_stack.yaml). Below are some things to look out for if you cannot find the cause. 

**Secrets**

- Check whether there is a a secret in the harbor-cluster namespace containing the AWS authentication for the the s3 bucket used for backups.

- Check if the secret is referenced in your pods definition. This can be configured in your fullstack for launching harbor under: storage -> s3.

- Check that the postgres-pod secret exists with the configurations for postgres correctly enabled. This exists as `pod-secret`.

- Check that secrets exist for: redis, postgres, core, jobservice, notaryserver, notarysigner, registry and trivy.

**Namespaces**

- Ensure that the `harbor-operator-app` is created in the `harbor-operator` namespace. This is because helm relies on this for creation of some resources.

- Equally, ensure that the components are being created in the `harbor-cluster` namespace. This is because it is hardcoded into a rolebinding for PodSecurityPolicies.

**Certificates**

- Check that certificates have been properly issued. In the fullstack.yaml the clusterissuer should be set to `letsencrypt-giantswarm`.

**Other**

- Check that the URL and DNS for core and notary are set correctly for the cluster you're working on in the fullstack.yaml.

- Check that the nginx ingress class has the following annotation:
  `ingressclass.kubernetes.io/is-default-class: "true"`

- Check that the nginx ingress is assigned to the core and notary dns.
```

kubectl get ingress -n harbor-cluster

kubectl edit ingress -n harbor-cluster harbor-cluster-harbor-habor

```
Under spec add: `ingressClassName: nginx`

- Clean up any crds or resources left over from a previous deployment.

- Check that your storage options have been configured correctly. Do they have enough memory? Are they pointing to the correct bucket? Do the credentials exist?
