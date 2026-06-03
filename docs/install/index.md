# Install 

There are a ton of options on doing an OpenShift install. This guide is focused on individuals and organizations who want to look at an enterprise view of the platform. Most organizations don't just run a single cluster - they usually run tens or hundreds of clusters. 

For the install, this documentation will focus on building a small hub and spoke cluster architecture model ([link](https://www.redhat.com/en/blog/using-red-hat-advanced-cluster-management-and-openshift-gitops-to-manage-openshift-virtualization)). The documentation also assumes we are installing in a bare metal, on-premise environment. This install process includes: 

* Gather the [prerequisites](prerequisites.md)
* Create the [installation host](installation-host.md)
    * Download all the necessary tools for the install
* Install the hub cluster
    * Create the hub cluster install configurations
    * Create the hub cluster using the OpenShift installer cli
* Bootstrap the hub cluster with Red Hat Advanced Cluster Management (ACM)
* Provide all the necessary host inventory or hyperscaler credentials
* Create the spoke cluster using ACM

After the install process is complete, the post install process can begin. 

## Acronyms

| Acronym | Definition                                              |
| ---     | ---                                                     |
| OCP     | OpenShift Container Platform                            |
| OVE     | OpenShift Virtualization Engine                         |
| OKE     | OpenShift Kubernetes Engine                             |
| OPP     | OpenShift Platform Plus                                 |
| SNO     | Single Node OpenShift                                   |
| ACM     | Red Hat Advanced Cluster Management for Kubernetes      |
| ODF     | OpenShift Data Foundation                               |