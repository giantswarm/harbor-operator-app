---

title: "Harbor Down"

---

This alert fires because a component of Harbor is not working as expected.

First step would be to check if the pod is up and what the logs for the pod says:

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

- Clean up any crds or resources left over from a previous deployment.

- Check that your storage options have been configured correctly. Do they have enough memory? Are they pointing to the correct bucket? Do the credentials exist?
