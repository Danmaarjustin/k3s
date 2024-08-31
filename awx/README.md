#Deploy AWX

Create dir(No need of you cloned the repo) and make namespace:

```bash
cd ./k3s/awx
mkdir installer
cd installer
```

Create namespace awx:
```bash
kubectl create namespace awx
```

Install the AWX operator:

Ctrl+c and Ctrl+v
```bash
helm install awx-operator awx-operator-2.19.1.tgz
```

#Create certs and create kubernetes secret with them:

```bash
cd ..
mkdir certs
cd certs
kubectl exec -it vault-0 -n vault -- vault write pki/issue/k3s common_name="awx.k3s.internal" alt_names="awx.k3s.internal" ttl="168h" > cert_output.json
cat cert_output.json
```

Copy and paste the certificate and private-key output of the cert_output.json into the correct files:
key in private_key.pem
cert in certificate.pem

Create secret based on input:

Ctrl+c and Ctrl+v:
```bash
kubectl create secret tls awx-k3s-internal \
 --cert=certificate.pem \
 --key=private_key.pem \
 -n awx
```

#Apply custom resource, service and ingress for awx:

```bash
cd ./k3s/awx/
kubectl apply -f awx-cr.yaml
kubectl apply -f awx-service.yaml
kubectl apply -f awx-ingress.yaml
```
When all containers are ready and deployed(This can take a couple of minutes):

Fetch password for innitial login:

Ctrl+c and Ctrl+v:
```bash
kubectl get secret -n awx awx-admin-password -o jsonpath="{.data.password}" | base64 --decode
```
Username: admin