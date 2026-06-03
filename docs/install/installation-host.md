# Building the OpenShift Installation Host

The installation host serves multiple purposes. All of our install work is done here. It's used to validate the environment prior to installation. It provides additional tools for installation such as a web server environment for the ISO images.

The installation host can be a bare metal or virtualized host and only requires what is needed for a normal Red Hat Enterprise Linux install. We recommend at least 2 CPU, 8 GB memory and 100 GB disk. It should be on the same network as the targeted hosts for the OpenShift cluster install so we can use it to validate the firewall is open prior to installation. 

> But I want to use a company image? Please make sure any processes that may interfere with web hosting or the other install processes is turned off. 

## Install the OS

1. Download Red Hat Enterprise Linux 9.x Binary DVD ([link](https://access.redhat.com/downloads/content/rhel))
2. Boot host from ISO and perform install as Server with GUI
3. Make sure to enable SSH for newly created user and make them an administrator
4. Reboot and SSH into installation host as administrative user

## Register the Host

To be able to install necessary packages, you need to register the host with your Red Hat Account ID and password. You could have also accomplished this during the RHEL install. 

```shell
sudo subscription-manager register # Enter Red Hat account username/password when prompted
sudo subscription-manager repos --enable=rhel-9-for-x86_64-baseos-rpms
sudo subscription-manager repos --enable=rhel-9-for-x86_64-appstream-rpms
sudo dnf update -y 
sudo reboot
```

## Download Required Tools

Install the required tools.

```shell
OCP_VERSION=latest
wget "https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/stable-${OCP_VERSION}/openshift-install-linux.tar.gz" -P /tmp
sudo tar -xvzf /tmp/openshift-install-linux.tar.gz -C /usr/local/bin
wget "https://mirror.openshift.com/pub/openshift-v4/clients/ocp/stable-${OCP_VERSION}/openshift-client-linux.tar.gz" -P /tmp
sudo tar -xvzf /tmp/openshift-client-linux.tar.gz -C /usr/local/bin
rm /tmp/openshift-install-linux.tar.gz /tmp/openshift-client-linux.tar.gz -y
sudo dnf install nmstate git podman 
```

Check for the availability of the required tools.

```shell
openshift-install version
oc version
nmstatectl -V
git -v
podman --version
```

## Download the Pull Secret

The pull secret is available at [https://console.redhat.com/openshift/install/pull-secret](https://console.redhat.com/openshift/install/pull-secret). Download the file to `~/.pull-secret`.

## Create an SSH Key

An SSH key is required to access the OpenShift hosts in cases of debugging requirements and is required as part of the OpenShift install. 

```shell
ssh-keygen -t ed25519 -f ~/.ssh/ocp_ed25519
```

## Open Port 8080 for iso-http

The web interfaces for the major BMCs can sometimes be flaky when uploading ISOs for boot. We would typically like to serve the ISO from an HTTP host. If you have an existing HTTP host, you will need to have permission to copy the ISO from here to there. If not, we can use Podman to run an HTTP web server locally. To do so, let's open the firewall on the host. 

```shell
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload
```

## Create DNS Entries

Create the following A records in your DNS based on the values from the [prerequisites](prerequisites.md).

| A Record                     | IP Address | Description                           |
| ---                          | ---        | ---                                   |
| `api.<cluster_suffix>`       | 10.1.0.9   | Virtual IP (VIP) for the API endpoint |
| `api-int.<cluster_suffix>`   | 10.1.0.9   | Internal VIP for the API endpoint     |
| `*.apps.<cluster_suffix>`    | 10.1.0.10  | Virtual IP for the ingress endpoint   |

Validate the DNS using dig:

```shell
dig +noall +answer @<dns> api.<cluster_suffix>
dig +noall +answer @<dns> test.apps.<cluster_suffix>
```

> [DNS requirements](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html-single/installing_on_bare_metal/index#network-requirements-dns_ipi-install-prerequisites)
> [Validating DNS resolution for user-provisioned infrastructure](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html-single/installing_on_bare_metal/index#installation-user-provisioned-validating-dns_installing-bare-metal-network-customizations)

#### Optional Helpful DNS Entries

| A Record                                   | IP Address | Description        |
| ---                                        | ---        | ---                |
| `installation.<cluster_suffix>`            | 10.1.0.4   | IP for the installation host |
| `openshift-control-plane-1.<cluster_suffix>` | 10.1.0.11  | IP for cp1         |
| `openshift-control-plane-2.<cluster_suffix>` | 10.1.0.12  | IP for cp2         |
| `openshift-control-plane-3.<cluster_suffix>` | 10.1.0.13  | IP for cp3         |
| `openshift-worker-1.<cluster_suffix>`      | 10.1.0.21  | IP for w1          |
| `openshift-worker-2.<cluster_suffix>`      | 10.1.0.22  | IP for w2          |
| `openshift-worker-3.<cluster_suffix>`      | 10.1.0.23  | IP for w3          |

## Firewall Checks

Documentation for the firewall configuration ([link](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/installation_configuration/configuring-firewall)). We will need access to the following for pulling OpenShift container images.

```
registry.redhat.io
access.redhat.com
registry.access.redhat.com
quay.io
cdn.quay.io
cdn01.quay.io
cdn02.quay.io
cdn03.quay.io
cdn04.quay.io
cdn05.quay.io
cdn06.quay.io
sso.redhat.com
```

Required for Telemetry:

```
cert-api.access.redhat.com
api.access.redhat.com
infogw.api.openshift.com
```

Other Content:

```
console.redhat.com 
*.apps.<cluster_name>.<base_domain>
api.openshift.com
mirror.openshift.com
quayio-production-s3.s3.amazonaws.com
rhcos.mirror.openshift.com
sso.redhat.com
storage.googleapis.com/openshift-release 
registry.connect.redhat.com
```

### MITM Proxy and Self-Signed Certificates

If you are using a MITM proxy which is doing TLS inspection, the Red Hat CDN for `dnf install` or `yum install` uses `cdn.redhat.com` which has a self-signed certificate. If your MITM proxy automatically blocks self-signed certificates, you will need to whitelist `cdn.redhat.com`.

It is recommended that all entries for the firewall as detailed above are also allowed through any TLS inspection processes.

### Connectivity Checks

Here's a small script for checking these:

```shell 
for domain in registry.redhat.io access.redhat.com registry.access.redhat.com quay.io cdn.quay.io cdn01.quay.io cdn02.quay.io cdn03.quay.io cdn04.quay.io cdn05.quay.io cdn06.quay.io sso.redhat.com cert-api.access.redhat.com api.access.redhat.com infogw.api.openshift.com console.redhat.com; do
  nc -zv -w 2 $domain 443 2>&1 | grep -iqE "connected|succeeded|open" && echo "$domain: SUCCESS" || echo "$domain: FAILED"
done
```

You can also use netcat to test port connectivity individually:

```shell
nc -zv registry.redhat.io 443
nc -zv sso.redhat.com 443
```

You can also login with your Red Hat account using podman:

```shell
podman login registry.redhat.io
```

## Tools for Environment Validation

Make sure you have all of these installed:

```shell
ping registry.redhat.io        # ICMP doesn't always work, but try
curl -vk https://registry.redhat.io/v2/
dig registry.redhat.io +short
nslookup registry.redhat.io
podman login registry.redhat.io
```

Now go [install](cluster-install.md).
