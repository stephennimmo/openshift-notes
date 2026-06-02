# Creating a Machine Configuration File with Butane

[Butane](https://coreos.github.io/butane/) is a tool that transpiles human-readable configs into MachineConfig resources. It avoids having to manually base64 encode file contents.

## Install Butane

```shell
curl -LO https://mirror.openshift.com/pub/openshift-v4/clients/butane/latest/butane
chmod +x butane
sudo mv butane /usr/local/bin/
```

## Write a Butane Config

Create a file called `99-worker-example.bu`:

```yaml
variant: openshift
version: {{ ocp_version }}.0
metadata:
  name: 99-worker-example
  labels:
    machineconfiguration.openshift.io/role: worker
storage:
  files:
    - path: /etc/file.conf
      mode: 0644
      overwrite: true
      contents:
        inline: |
          mysecretvalue
```

The `inline` field accepts plain text directly — no base64 encoding required. Butane handles the encoding during transpilation.

## Transpile to a MachineConfig

```shell
butane 99-worker-example.bu -o 99-worker-example.yaml
```

This produces a standard MachineConfig resource with the file contents base64 encoded automatically.

## Using a Local File Instead of Inline

You can reference a local file rather than inlining the contents:

```yaml
variant: openshift
version: {{ ocp_version }}.0
metadata:
  name: 99-worker-example
  labels:
    machineconfiguration.openshift.io/role: worker
storage:
  files:
    - path: /etc/file.conf
      mode: 0644
      overwrite: true
      contents:
        local: file.conf
```

Then pass `--files-dir` when transpiling:

```shell
butane 99-worker-example.bu --files-dir . -o 99-worker-example.yaml
```

## Apply the MachineConfig

```shell
oc apply -f 99-worker-example.yaml
```
