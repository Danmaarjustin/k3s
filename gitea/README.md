#GITEA DEPLOYMENT

Create namespace gitea
Ctrl+c and Ctrl+v:
```bash
kubectl create namespace gitea
```
#Create certificate

Ctrl+c and Ctrl+v:
```bash
cd gitea
mkdir certs
cd certs
touch certificate.pem private_key.pem
kubectl exec -it vault-0 -n vault -- vault write pki/issue/k3s common_name="gitea.k3s.internal" alt_names="gitea.k3s.internal" ttl="168h" > cert_output.json
cat cert_output.json
```
Copy and paste the correct certificate in certificate.pem and copy the correct key in private_key.pem.

Create certificates with vault:

Ctrl+c and Ctrl+v:
```bash
kubectl create secret tls gitea-k3s-internal \
 --cert=certificate.pem \
 --key=private_key.pem \
 -n gitea
```
#Add repo and install gitea

Ctrl+c and Ctrl+v:
```bash
helm repo add gitea-charts https://dl.gitea.com/charts/
helm repo update
helm install gitea gitea-charts/gitea -n gitea
```
When deployement is ready apply the ingress:

Check if ready:
Ctrl+c and Ctrl+v:
```bash
kubectl get pods -n gitea
```

Apply gitea ingress:

Ctrl+c and Ctrl+v:
```bash
apply -f gitea-ingress.yaml
```

#Get pod information to change password via pod cli:

Ctrl+c and Ctrl+v:
```bash
kubectl get pods -n gitea
kubectl exec -it -n gitea gitea-66b98969bb-6ntgq -- gitea admin user change-password -u gitea_admin -p changeme
