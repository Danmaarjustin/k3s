########################## AWX

cd /home/sysadmin/k3s/awx
mkdir installer
cd installer
k create namespace awx
helm install awx-operator awx-operator-2.19.1.tgz

cd ..

cat <<EOF > awx-cr.yaml
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
  namespace: awx
spec:
  service_type: ClusterIP
  ingress_type: ingress
  hostname: awx.k3s.internal
EOF

cat <<EOF > awx-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: awx-ingress
  namespace: awx
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: "websecure"
    traefik.ingress.kubernetes.io/router.tls: "true"
    # Remove the middleware annotation if not used
    # traefik.ingress.kubernetes.io/router.middlewares: "awx@kubernetescrd"
spec:
  tls:
  - hosts:
    - awx.k3s.internal
    secretName: awx-tls-secret
  rules:
  - host: awx.k3s.internal
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: awx-service
            port:
              number: 80
EOF

cat <<EOF > awx-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: awx-service
  namespace: awx
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 8052
  selector:
    app.kubernetes.io/component: awx
    app.kubernetes.io/name: awx-web
EOF


mkdir certs
cd certs
kubectl exec -it vault-0 -n vault -- vault write pki/issue/k3s common_name="awx.k3s.internal" alt_names="awx.k3s.internal" ttl="168h" > cert_output.json

cat cert_output.json
kubectl create secret tls awx-k3s-internal \
 --cert=certificate.pem \
 --key=private_key.pem \
 -n awx

cd ..

When all containers are ready and deployed:

kubectl get secret -n awx awx-admin-password -o jsonpath="{.data.password}" | base64 --decode
to get the admin password for innitial login