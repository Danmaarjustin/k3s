##################ArgoCD

helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm install argocd argo/argo-cd --namespace argocd --create-namespace

when deployment is ready and all pods running:
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
Jcjoq6hQx4brtK50
username: admin


mkdir certs
cd certs
touch certificate.pem private_key.pem

kubectl exec -it vault-0 -n vault -- vault write pki/issue/k3s common_name="argocd.k3s.internal" alt_names="argocd.k3s.internal" ttl="168h" > cert_output.json
cat cert_output.json

kubectl create secret tls argocd-k3s-internal \
 --cert=certificate.pem \
 --key=private_key.pem \
 -n argocd


cat <<EOF > argocd-ingressRoute.yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: argocd-https
  namespace: argocd
spec:
  entryPoints:
    - websecure
  routes:
  - kind: Rule
    match: Host(`argocd.k3s.internal`)
    priority: 10
    services:
      - name: argocd-server
        port: 80
  - kind: Rule
    match: Host(`argocd.k3s.internal`) &&  Header(`Content-Type`, `application/grpc`)
    priority: 11
    services:
      - name: argocd-server
        port: http
        scheme: h2c
  tls:
    secretName: argocd-k3s-internal

EOF

kubectl edit configmap argocd-cmd-params-cm -n argocd
set: server.insecure: "true"
#Delete argocd server pod:
kubectl delete pod -l app.kubernetes.io/name=argocd-server -n argocd

kubectl apply -f argocd-ingressRoute.yaml