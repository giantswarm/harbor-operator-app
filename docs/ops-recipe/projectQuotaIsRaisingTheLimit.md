---

title: "ProjectQuotaIsRaisingTheLimit"

---

This alert fires because a Harbor project is nearing the limit of its capacity.

Projects are created with a default capacity value of `-1` which means it has unlimited storage. If your projects are continuously approaching their limit, you may want to consider switching the limit to this.

The alert should show the name of the project that is getting full. You can manage and adjust the projects capacity in the Harbor UI. 


- Select Project Quotas from the adminstrative tabs on the Harbor UI.

- Click the box next to the project that is nearing capacity.

- Select edit from the text above and change the capacity to suit your needs.

More information can be found [here](https://goharbor.io/docs/2.6.0/administration/configure-project-quotas/) on Harbors official documentation.
