# ZABBIX

Reference to helm chart for zabbix:
```bash
https://artifacthub.io/packages/helm/zabbix-community/zabbix
```

##Add helm repo and update:
Ctrl+c and Ctrl+v:
```bash
helm repo add zabbix-community https://zabbix-community.github.io/helm-zabbix
helm repo update
```

Check for version:
Ctrl+c and Ctrl+v:
```bash
helm search repo zabbix-community/zabbix -l
```

Export latest version into env variable:

Ctrl+c and Ctrl+v:
```bash
export ZABBIX_CHART_VERSION='5.0.1'
```
Get zabbix helm chart and save as "zabbix_values.yaml"

Ctrl+c and Ctrl+v:
```bash
helm show values zabbix-community/zabbix --version $ZABBIX_CHART_VERSION > $HOME/zabbix_values.yaml
```
Edit the file to youre needs and install or dry-run the install:

Ctrl+c and Ctrl+v:
```bash
helm upgrade --install zabbix zabbix-community/zabbix \
 --dependency-update \
 --create-namespace \
 --version $ZABBIX_CHART_VERSION \
 -f $HOME/zabbix_values.yaml -n zabbix --debug --dry-run
```
Remove --dry-run to make kubernetes install

#Create a certificate and store into files:

Ctrl+c and Ctrl+v:
```bash
cd ./zabbix
mkdir certs
cd ./certs
kubectl exec -it vault-0 -n vault -- vault write pki/issue/k3s common_name="zabbix.k3s.internal" alt_names="zabbix.k3s.internal" ttl="168h" > cert_output.json
touch certificate.pem private_key.pem
cat cert_output.json
```
Copy and paste the correct certificate in certificate.pem and copy the correct key in private_key.pem.

##Make kubernetes secret:

Ctrl+c and Ctrl+v:
```bash
kubectl create secret tls zabbix-k3s-internal \
 --cert=certificate.pem \
 --key=private_key.pem \
 -n zabbix
```

##Apply ingress now certificate is created:

Ctrl+c and Ctrl+v:
```bash
cd ../
kubectl apply -f zabbix-ingress.yaml
```


Login to Zabbix:

URL: http://zabbix.k3s.internal
Login: Admin
Password: zabbix



