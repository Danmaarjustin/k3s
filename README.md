##############Install master node
su
apt install curl git gpg ca-certificates nfs-common -y
hostnamectl set-hostname k-master
echo k-master > /etc/hostname
curl -sfL https://get.k3s.io | sh -

sudo cat /var/lib/rancher/k3s/server/node-token


#Mount nfs synology see ----> https://youtu.be/uPt3VKQOMBs?si=Qk2-f3jG61OQjdfr
/mnt #for example
mkdir /mnt
mount -t nfs <IP-ADDRESS-SYNOLOGY>:volume1/<NAME-OF-SHARE> /mnt




	OUTPUT:
K10acf961dac4ff6d4968497aced3f225d4663aad146bf313db746c0a816ed59418::server:391bad00f60f3fb2998e1739b3ec968f

echo "alias k=$(which kubectl)" >> ~/.bashrc
or
echo "alias k=/usr/local/bin/kubectl" >> ~/.bashrc

mkdir -p ~/.kube
cp /etc/rancher/k3s/k3s.yaml ~/.kube/config

chown $(whoami):$(whoami) ~/.kube/config
or
echo "export KUBECONFIG=~/.kube/config" >> ~/.bashrc

source ~/.bashrc

mkdir k3s/rancher
mkdir k3s/awx
mkdir k3s/gitea
mkdir k3s/vault
mkdir k3s/zabbix
mkdir k3s/traefik
mkdir k3s/argocd

#Install helm

curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh

or # Only for Debian and Ubuntu 

curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm

##############Install worker node
su
hostnamectl set-hostname k-worker1
echo k-worker1 > /etc/hostname
curl -sfL https://get.k3s.io | K3S_URL=https://10.10.10.31:6443 K3S_TOKEN=K10acf961dac4ff6d4968497aced3f225d4663aad146bf313db746c0a816ed59418::server:391bad00f60f3fb2998e1739b3ec968f sh -
hostnamectl set-hostname k-worker1
echo k-worker1 > /etc/hostname

Repeat steps for more than 1 worker




####################### DEPLOY RANHCER

mkdir certs
cd certs
touch certificate.pem private_key.pem

kubectl create namespace cattle-system

kubectl exec -it vault-0 -n vault -- vault write pki/issue/k3s common_name="rancher.k3s.internal" alt_names="rancher.k3s.internal" ttl="168h" > cert_output.json
cat cert_output.json

kubectl create secret tls rancher-k3s-internal \
 --cert=certificate.pem \
 --key=private_key.pem \
 -n cattle-system


kubectl -n cattle-system create secret generic tls-ca   --from-file=ca.crt

helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo update


helm install rancher rancher-latest/rancher   --namespace cattle-system   --set hostname="rancher.k3s.internal"   --set ingress.tls.source=secret

	OUTPUT:
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

```
echo https://rancher.k3s.internal/dashboard/?setup=$(kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}')
```

To get just the bootstrap password on its own, run:

```
kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}{{ "\n" }}'
```


Happy Containering!


#######################################

kubectl edit ingress -n cattle-system rancher

look for this:
tls:
- hosts:
  - rancher.k3s.internal
  secretName: rancher-k3s-internal


	WHEN EVERYTHING IS RUNNING:

kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}{{ "\n" }}'

THIS RETURNS A KEY NEEDED FOR INNITIAL LOGIN

Go to web gui and paste


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
############################ DEPLOY ZABBIX

# ZABBIX

https://artifacthub.io/packages/helm/zabbix-community/zabbix

helm repo add zabbix-community https://zabbix-community.github.io/helm-zabbix
helm repo update

helm search repo zabbix-community/zabbix -l

	LATEST VERSION ATM
export ZABBIX_CHART_VERSION='5.0.1'

helm show values zabbix-community/zabbix --version $ZABBIX_CHART_VERSION > $HOME/zabbix_values.yaml

#Remove --dry-run to make kubernetes install
helm upgrade --install zabbix zabbix-community/zabbix \
 --dependency-update \
 --create-namespace \
 --version $ZABBIX_CHART_VERSION \
 -f $HOME/zabbix_values.yaml -n zabbix --debug --dry-run


kubectl exec -it vault-0 -n vault -- vault write pki/issue/k3s common_name="zabbix.k3s.internal" alt_names="zabbix.k3s.internal" ttl="168h" > cert_output.json

touch certificate.pem private_key.pem

kubectl create secret tls zabbix-k3s-internal \
 --cert=certificate.pem \
 --key=private_key.pem \
 -n zabbix

cat <<EOF > zabbix-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: zabbix-web-ingress
  namespace: zabbix
spec:
  tls:
  - hosts:
    - zabbix.k3s.internal
    secretName: zabbix-k3s-internal
  rules:
  - host: zabbix.k3s.internal
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: zabbix-zabbix-web
            port:
              number: 80
EOF

kubectl apply -f zabbix-ingress.yaml

Login to Zabbix:

URL: http://zabbix.k3s.internal
Login: Admin
Password: zabbix


kubectl exec -it vault-0 -n vault -- vault write pki/issue/k3s common_name="zabbix.k3s.internal" alt_names="zabbix.k3s.internal" ttl="168h" > cert_output.json

touch certificate.pem private_key.pem

kubectl create secret tls zabbix-k3s-internal \
 --cert=certificate.pem \
 --key=private_key.pem \
 -n zabbix



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










