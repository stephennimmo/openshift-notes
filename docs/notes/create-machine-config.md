# Creating a Machine Configuration File

How to base64 encode a string:

```shell
echo -n "mysecretvalue" | base64
```

To decode:

```shell
echo -n "bXlzZWNyZXR2YWx1ZQ==" | base64 -d
```

How to encode a file:

```shell
# encode
base64 -w0 /path/to/file
# decode
base64 /path/to/file > file.b64
# decode
base64 -d file.b64 > file.decoded
```

Then you take the base64 encoded value and paste it into the file after the `base64,`

```yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  name: 99-worker-example
  labels:
    machineconfiguration.openshift.io/role: worker
spec:
  config:
    ignition:
      version: 3.4.0
    storage:
      files:
      - path: /etc/file.conf
        mode: 0644
        overwrite: true
        contents:
          source: data:text/plain;charset=utf-8;base64,bXlzZWNyZXR2YWx1ZQ==
```
