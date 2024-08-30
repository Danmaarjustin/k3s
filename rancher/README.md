#DEPLOY RANHCER

##Prepperation

Add directorys

Ctrl+c and Ctrl+v:
```bash
mkdir ./rancher/certs
cd certs
touch certificate.pem private_key.pem ca.crt
```

Create namespace in kubernetes:

Ctrl+c and Ctrl+v:
```bash
kubectl create namespace cattle-system
```

Create certificate by hand and paste output in cert_output.json:

Ctrl+c and Ctrl+v:
```bash
kubectl exec -it vault-0 -n vault -- vault write pki/issue/k3s common_name="rancher.k3s.internal" alt_names="rancher.k3s.internal" ttl="168h" > cert_output.json
cat cert_output.json
```
Copy output in correct certificate and private-key file.
When this is done make certificate based on these files:

Ctrl+c and Ctrl+v:
```bash
kubectl create secret tls rancher-k3s-internal \
 --cert=certificate.pem \
 --key=private_key.pem \
 -n cattle-system
```

Create a seperate ca secret, which rancher needs to deploy:

Copy the ca_chain part(The first certificate) into ca.crt.

Example input:
```console
-----BEGIN CERTIFICATE-----
MIIDODCCAiCgAwIBAgIUBBp9AXwqtPp4fHZz3NKWD9F3IrcwDQYJKoZIhvcNAQEL
BQAwFzEVMBMGA1UEAxMMazNzLmludGVybmFsMB4XDTI0MDgyOTE3MzgxN1oXDTI0
MDkzMDE3Mzg0N1owFzEVMBMGA1UEAxMMazNzLmludGVybmFsMIIBIjANBgkqhkiG
9w0BAQEFAAOCAQ8AMIIBCgKCAQEAyLKVmC2UpJ5mwqhnLah7IeZPDNuVZQR3RWIR
AOyc/auP37hQ0TQPyLJyMrO1Wyz2Mb5aX0bNLIDnafWSaKAWySSQwQrXySPfaI5S
+jzJP0/W7opB6sCIMO/j7okODEu7X/5r07WAbnAgFglX26UziK03yiXb87ypMudQ
oWQbsVfCQYm9q+nYh1afHdn5MkHaU/g6gCgC+my7zPV0rFjQO7QQRjLX/ZzqHc6f
w6yEleM2JnmE9QanPasA6UtfPdwmNp1aBPaJxMKPlRpN0rX3aVcAK4uFHYDQOfZb
wwxgtkKQbO3RQ9qTZqvvo2bwojCGlRHJzeTrOG5Y6Qr2njzNEQIDAQABo3wwejAO
BgNVHQ8BAf8EBAMCAQYwDwYDVR0TAQH/BAUwAwEB/zAdBgNVHQ4EFgQU6evWtGhZ
T4axWT9A8bASHQRbzfcwHwYDVR0jBBgwFoAU6evWtGhZT4axWT9A8bASHQRbzfcw
FwYDVR0RBBAwDoIMazNzLmludGVybmFsMA0GCSqGSIb3DQEBCwUAA4IBAQCS8u3c
BClpZ1cajxxWoBIHTDQKRwN3TuHHD5//n1O2NL6NKyDg1DOq9nya6pTg+rUrNUBb
CpxGY+iod5TN8WBGFCH8mtOTR8o5mYZAcH/KIKGBc9jVUZ5JqTZ/G8E9MiYwgHo8
T3n4+tQTmNw6kw+aNxGdEjpy3yp/2kOrBRkDfEePloAuBfkLmBIeoYsIiTic2tV/
C5/o2tLz7DENZfPEk59Xe73+BtWfIfIy7itKQlVbQdvEyuS94031Ug6zzdF/W5+n
e9aSR1SY+0Aq7iB9d7Sn5pCENfMQXTAQa1rvQCbnEwkupGIBxtXzTiOXINQ7od7o
CY4kn0njddeWDsH2
-----END CERTIFICATE-----
```

Create the kubernetes ca certificate secret:

Ctrl-c and Ctrl+v:
```bash
kubectl -n cattle-system create secret generic tls-ca   --from-file=ca.crt
```

#Add repo to helm and update:

Ctrl+c and Ctrl+v:
```bash
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo update
```
#Install rancher
```bash
helm install rancher rancher-latest/rancher   --namespace cattle-system   --set hostname="rancher.k3s.internal"   --set ingress.tls.source=secret
```

Helm install output should say something like:
OUTPUT:
```console
NAME: rancher
LAST DEPLOYED: Thu Aug 22 00:15:55 2024
NAMESPACE: cattle-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Rancher Server has been installed.

NOTE: Rancher may take several minutes to fully initialize. Please standby while Certificates are being issued, Containers are started and the Ingress rule comes up.

Check out our docs at https://rancher.com/docs/

If you provided your own bootstrap password during installation, browse to https://rancher.k3s.internal to get started.

If this is the first time you installed Rancher, get started by running this command and clicking the URL it generates:

---
echo https://rancher.k3s.internal/dashboard/?setup=$(kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}')
---

To get just the bootstrap password on its own, run:

---
kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}{{ "\n" }}'
---


Happy Containering!
```

#Edit the automatically created ingress for rancher

Ctrl-c and Ctrl+v:
```bash
kubectl edit ingress -n cattle-system rancher
```

look for this:
```console
tls:
- hosts:
  - rancher.k3s.internal
  secretName: rancher-k3s-internal
```
Change the "secretName:" to:
```bash
rancher-k3s-internal
```

Check regulary if deployment is finished by checken all the pods for the namespace cattle-system by doing:

Ctrl-c and Ctrl-v:
```bash
watch kubectl get pods -n cattle-system
```

When it running check for innitial password for admin user:

Ctrl+c and Ctrl+v:
```bash
kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}{{ "\n" }}'
```

##Go to web gui and paste, change password and youre good to go.

