#ArgoCD Deployment

#Update helm repo:

Ctrl+c and Ctrl+v:
```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```

Install argocd via helm:

Ctrl+c and Ctrl+v:
```bash
helm install argocd argo/argo-cd --namespace argocd --create-namespace
```

##When deployment is ready and all pods running:
This can be checked by doing:
Ctrl+c and Ctrl+v:
```bash
kubectl get pods -n argocd
```

To get the secret for the admin account:

Ctrl+c and Ctrl+v:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```
OUTPUT:
```console
Jcjoq6hQx4brtK50
```
Username for argocd: admin

But first create folders for certificate and key:

Ctrl+c and Ctrl+v:
```bash
cd argocd
mkdir certs
cd certs
touch certificate.pem private_key.pem
```

Create certificate and key:

Ctrl+c and Ctrl+v:
```bash
kubectl exec -it vault-0 -n vault -- vault write pki/issue/k3s common_name="argocd.k3s.internal" alt_names="argocd.k3s.internal" ttl="168h" > cert_output.json
cat cert_output.json
```

Create sercret with key and certificate:

Ctrl+c and Ctrl+v:
```bash
kubectl create secret tls argocd-k3s-internal \
 --cert=certificate.pem \
 --key=private_key.pem \
 -n argocd
```

#Configmap edit by ignoring the warning for no secure connection from the ingress controller to the gitea webserver

Ctrl+c and Ctrl+v:
```bash
kubectl edit configmap argocd-cmd-params-cm -n argocd
```

Search for below part and set from false to true:
```console
set: server.insecure: "true"
```


#Delete argocd server pod to get the configmap ready:
Ctrl+c and Ctrl+v:
```bash
kubectl delete pod -l app.kubernetes.io/name=argocd-server -n argocd
```

Finaly apply the ingressRoute for gitea:

Ctrl+c and Ctrl+v:
```bash
kubectl apply -f argocd-ingressRoute.yaml
```