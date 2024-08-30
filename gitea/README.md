################################GITEA

k create namespace gitea
touch certificate.pem private_key.pem
kubectl exec -it vault-0 -n vault -- vault write pki/issue/k3s common_name="gitea.k3s.internal" alt_names="gitea.k3s.internal" ttl="168h" > cert_output.json
kubectl create secret tls gitea-k3s-internal \
 --cert=certificate.pem \
 --key=private_key.pem \
 -n gitea


cat <<EOF > gitea-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gitea-ingress
  namespace: gitea
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: "websecure"
    traefik.ingress.kubernetes.io/router.tls: "true"
    # Remove the middleware annotation if not used
    # traefik.ingress.kubernetes.io/router.middlewares: "awx@kubernetescrd"
spec:
  tls:
  - hosts:
    - gitea.k3s.internal
    secretName: gitea-k3s-internal
  rules:
  - host: gitea.k3s.internal
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: gitea-http
            port:
              number: 3000
EOF
helm repo add gitea-charts https://dl.gitea.com/charts/
helm repo update
helm install gitea gitea-charts/gitea -n gitea

apply -f gitea-ingress.yaml

k get pods -n gitea
k exec -it -n gitea gitea-66b98969bb-6ntgq -- gitea admin user change-password -u gitea_admin -p changeme
