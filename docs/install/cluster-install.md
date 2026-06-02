# OpenShift Install on Bare Metal in your datacenter

Did you already perform the [prerequisites](prerequisites.md)? 
This should all be done from the [bastion](bastion.md). 

## Create the Configurations

The typical agent based install requires two configuration files.

- install-config.yaml
- agent-config.yaml

The install-config.yaml contains all the cluster level information while the agent-config.yaml provides host level configurations.

> You can accomplish all of this with bastion tools like vi. It is highly recommended to use tools like Visual Studio Code (vscode) to help with yaml formatting.

### Create the Working Directory

You are going to want to create a working directory and initialize it into a git repository.

```shell
mkdir -p ocp && cd ocp
git init 
echo "install/" > .gitignore
echo "# POC install notes" > notes.md
touch install-config.yaml
touch agent-config.yaml
git add -A
git commit -m "repo initialized"
```

### Create the Install Config

Here is an example of the install-config.yaml contents.

```yaml
apiVersion: v1
baseDomain: ocp.basedomain.com
metadata:
  name: poc
controlPlane:
  name: master
  architecture: amd64
  hyperthreading: Enabled
  replicas: 3
compute:
- name: worker
  architecture: amd64
  hyperthreading: Enabled
  replicas: 3
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineNetwork:
  - cidr: 10.1.0.0/24
  networkType: OVNKubernetes
  serviceNetwork:
  - 172.30.0.0/16
platform:
  baremetal:
    apiVIP: 10.1.0.9
    ingressVIP: 10.1.0.10
    additionalNTPServers: 
      - 0.us.pool.ntp.org
      - 1.us.pool.ntp.org
pullSecret: 'value from ~/.pull_secret'
sshKey: 'value from ~/.ssh/ocp_ed25519.pub'
```

> If your environment requires a proxy to access the internet then copy, modify and append to the end of the install-config.yaml

```yaml
proxy:
  httpProxy: http://user:password@proxy.example.com:3128
  httpsProxy: http://user:password@proxy.example.com:3128
  noProxy: basedomain.com,localhost,127.0.0.1,.cluster.local,.svc,10.0.0.0/8,192.168.0.0/16,172.16.0.0/12,<your node subnet>,<api VIP>,<ingress VIP>
```

> If your environment is using a proxy, you may need to add the proxy's root authority as an additional trust bundle. Modify and append to the end of the install-config.yaml. 

```yaml
additionalTrustBundlePolicy: Always
additionalTrustBundle: | 
    -----BEGIN CERTIFICATE-----
  MIIDzTCCArWgAwIBAgIUXXXXXXXXXXXXXXXXXXXXXXXXXXXwDQYJKoZIhvcNAQEL
  ...
  -----END CERTIFICATE-----
  -----BEGIN CERTIFICATE-----
  ...
  -----END CERTIFICATE-----
```

### Create the Agent Config

The agent-config.yaml is basically a list of hosts' information. You'll create a list item for each of the hosts. Here's an example of a host with two ethernet connections bonded together in an LACP bond. 

```yaml
apiVersion: v1alpha1
kind: AgentConfig
metadata:
  name: poc
rendezvousIP: 10.1.0.11                     # IP of the first control-plane1 host
additionalNtpSources:
  - 0.us.pool.ntp.org
  - 1.us.pool.ntp.org
hosts:
  - hostname: ocp-poc-cp-01                 # hostname
    role: master                            # master/worker
    rootDeviceHints:
      deviceName: "/dev/sda"                # Disk Hint
    interfaces:
      - name: eno1                          # Interface Name 1
        macAddress: A1:B2:3C:4D:1E:11       # Mac Address 1
      - name: eno2                          # Interface Name 2
        macAddress: A1:B2:3C:4D:2E:11       # Mac Address 2
    networkConfig:
      interfaces:
        - name: bond0
          type: bond
          state: up
          link-aggregation:
            mode: 802.3ad                   # LACP
            port:
              - eno1
              - eno2
            options:
              miimon: "100"
              lacp_rate: fast
          ipv4:
            enabled: false
          ipv6:
            enabled: false
        - name: bond0.3
          type: vlan
          state: up
          vlan:
            base-iface: bond0
            id: 3
          ipv4:
            enabled: true
            address:
              - ip: 10.1.0.11
                prefix-length: 24
            dhcp: false
          ipv6:
            enabled: false
      dns-resolver:
        config:
          server:
            - dns1.basedomain.com           # DNS 1
            - dns2.basedomain.com           # DNS 2
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 10.1.0.1      # Gateway for Machine Network
            next-hop-interface: bond0.3
            table-id: 254
```

And then do that for the other machines in the control plane and the workers. 

!!! note
    Notice the inconsistent labels and spellings. macAddress in the interfaces stanza, but mac-address in the networkConfig stanza. additionalNtpSources is used here, but it's additionalNTPServers in the install-config.yaml.

[Docs: Sample Config Bonds Vlans](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/installing_an_on-premise_cluster_with_the_agent-based_installer/preparing-to-install-with-agent-based-installer#agent-install-sample-config-bonds-vlans_preparing-to-install-with-agent-based-installer)

## Generate the ISO

From the ~/ocp directory, you can now create the agent iso. We create an install directory and copy the yaml files into the directory because the image creation process consumes and destroys the configuration files. We want to keep a copy in case the process needs to be repeated.

Here's an example script. I would recommend creating a small file named `touch create-iso.sh && chmod +x create-iso.sh` in the ocp working directory with the contents.

```shell
#!/bin/bash
rm -rf install
mkdir install
cp -r install-config.yaml agent-config.yaml install
openshift-install agent create image --dir=install
```

> You can always add --log-level=debug to any openshift-install commands for more output.

This is going to generate an agent.x86_64.iso file in the install directory which should be located at ~/ocp/install. Remember, everything in the install directory has been excluded from the git repository. 

## Host the ISO

The best way to provide the ISO as an HTTP source if you are using a BMC like iLo or iDRAC. Not doing so could result in issues during the install due to network contention using the web interfaces of those tools.

Luckily, on the bastion host, we already have podman installed so we can just use that to serve.

```shell
podman run -d --name iso-http \
  -p 8080:8080 \
  -v ~/ocp/install/agent.x86_64.iso:/var/www/html/agent.x86_64.iso:Z \
  registry.redhat.io/rhel9/httpd-24:9.6
```

Test retrieving the iso using the following command.

```shell
wget http://bastionhostname:8080/agent.x86_64.iso
```

> Don't forget to [open the host firewall](bastion.md#open-port-8080-for-iso-http) on the bastion.

If for some reason this solution will not work, you can install httpd and serve from there.

You can also copy it somewhere else from where the BMC has better access. Here's an example using scp.

```shell
scp user@192.168.122.187:~/ocp/install/agent.x86_64.iso ~/iso/
```

## Install the Cluster

When all the hosts are booted, wait for the bootstrap to complete.

```shell
openshift-install agent wait-for bootstrap-complete --dir=install
```

When the bootstrap is complete, wait for the install to complete.

```shell
openshift-install agent wait-for install-complete --dir=install
```

At the end of the process, you will be presented with the URL for the cluster endpoint, along with the kubeadmin credentials. They are also available in the install folder at ~/ocp/install as files kubeadmin-password and kubeconfig.

Open the URL presented and log in. Wait until all checks are green before proceeding. 

## Validate the Install

Login to the Cluster:

```shell
oc login --server=https://api.cluster.basedomain.com:6443 -u kubeadmin -p <password>
```

### Test Connectivity

```shell
oc debug node/<worker-node-name> -- chroot /host \
  podman pull registry.redhat.io/ubi9/ubi:latest
```

Cleanup the leftover install and configuration pods:

```shell
oc delete pods --all-namespaces --field-selector=status.phase=Succeeded
oc delete pods --all-namespaces --field-selector=status.phase=Failed
```

### Install Debugging

#### Booting in Debug Mode

Modify the Boot Parameters (GRUB/Boot Menu)

This is often the most reliable way to get a shell or verbose output when direct TTY switching fails. You'll need to intercept the boot process.

1. Reboot the machine with the ISO.
2. At the GRUB (bootloader) menu: As soon as you see the initial boot menu, press an arrow key (up/down) to stop the automatic countdown.
3. Edit the boot entry: Select the installer's default boot entry (usually the first one). Press the `e` key to edit the boot parameters.
4. Locate the `linux` or `linuxefi` line: This line contains the kernel arguments. Add debug/shell parameters to force an emergency shell. Go to the end of the `linux` or `linuxefi` line and add the param (see choices below). This will drop you into an initramfs shell before the root filesystem is mounted. It's a very minimal environment but allows `ip a show`, `dmesg`, and looking at files in the initramfs.
5. Press `Ctrl+X` or `F10` to boot with the modified parameters.

The three main alternatives are:

**`rd.break`**

How it works: This parameter tells dracut (the initramfs environment) to interrupt the normal boot process before the real root filesystem is mounted and before systemd takes over. You are dropped into an initramfs shell where you can manually mount /sysroot and chroot into it to make changes. Since this is very early in the boot sequence, the environment is limited, but it lets you work on the system before anything else initializes.

When to use it: This is the method of choice for situations where you need to fix problems directly on the root filesystem, especially when you can't log in normally—such as resetting a forgotten root password or repairing critical files before systemd runs. It's lower-level than systemd.unit=emergency.target and gives you control before the system even mounts root.

**`systemd.unit=emergency.target`**

How it works: This parameter tells the systemd process to boot directly into a minimal shell. It will try to mount the root filesystem as read-only and start only the most essential services required for an emergency shell. This is often the preferred method for general system troubleshooting.

When to use it: This is a good choice for fixing issues that aren't related to the initial filesystem or the boot process itself, such as a corrupt /etc/fstab or a misconfigured service. It gives you a more complete environment than rd.break but still keeps things simple.

**`init=/bin/bash`**

How it works: This is the most direct and basic method. It tells the kernel to bypass all the normal boot processes and execute /bin/bash directly as the first process (PID 1). This gives you a shell with no services, no network, and the root filesystem mounted as read-only.

When to use it: Use this as a last resort when other methods fail. It's the most primitive and powerful method, as it gives you control before any other processes or services start. It's ideal for a corrupted boot process or severe filesystem issues where rd.break or emergency.target might fail to load. It also requires you to manually remount the root filesystem as read-write, just like you were doing with rd.break.

## Issues that come up

* [Installing an OpenShift 4 cluster with the Agent-based Installer fail with "tls: failed to verify certificate: x509: certificate signed by unknown authority"](https://access.redhat.com/solutions/7070904). This usually happens when you are using some sort of web proxy and the certificate being presented for proxied connections was an intermediate certificate from the proxy. You need to add THAT root certificate and intermediaries to the installer to be able to trust the cert. 
