# Mark the UI as a POC Environment

Put a banner at the top to ensure everyone knows what environment they are in. 

```yaml
cat <<EOF | oc apply -f -
apiVersion: console.openshift.io/v1
kind: ConsoleNotification
metadata:
  name: poc-notification
spec:
  text: poc
  location: BannerTop 
  color: '#000000'
  backgroundColor: '#ff8330'
EOF
```
