#DEPLOY VAULT

##Deploy cert-manager Custom Resource Definition, extension for the kubernetes API needed for managing certificates

Ctrl+c en Ctrl+v
```bash
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.11.0/cert-manager.yaml
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.12.0/cert-manager.crds.yaml
```

##Add hashicorp helm repo and update

Ctrl+c en Ctrl+v
```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update
```

##Create namespace and install hashicorp vault

below will install hashicorp vault with helm in the namespace "vault".
The install will start the deployment in kubernetes

Ctrl+c en Ctrl+v
```bash
kubectl create namespace vault
helm install vault hashicorp/vault --namespace vault
```

#Innitial vault configuration
It can take a couple of seconds before the pods are ready
So we can check this by doing:

Ctrl+c en Ctrl+v
```bash
kubectl get pods -n vault vault-0
```
Status should be Running

OUTPUT:
```console
NAME                                    READY   STATUS    RESTARTS      AGE
vault-0                                 0/1     Running   1 (61s ago)   19h
vault-agent-injector-7bcc447788-2qwbm   1/1     Running   1 (61s ago)   19h
```
When pods are running, you can start configuring vault.

Init the vault operator by doing the following:

```bash
kubectl exec -it vault-0 -n vault -- vault operator init
```

EXAMPLE OUTPUT:
```console
Unseal Key 1: GZsBfHCqzoculkpqyLWxplAh3J4oVZ1lPzPDJk7D6YK9
Unseal Key 2: 0pBUSfF2y8IZcrEJuVTgPYPdkeKE/yjrVEFBsiJSUFkw
Unseal Key 3: QRRB5Ihs6rI0WJDLjH3HV0iN/vxeA1LNbxnuPh6ZcqW5
Unseal Key 4: 7OCtKumVLoEE5LTMdud/Or6Iq6upmhpHavasseTvexTh
Unseal Key 5: RnacJwch9F9/1OtTlJeEbYSykLwReCkw7rNsOJoJUYoc

Initial Root Token: hvs.iZaGP4yIDxUSMqkaaqZ1XOJZ

Vault initialized with 5 key shares and a key threshold of 3. Please securely
distribute the key shares printed above. When the Vault is re-sealed,
restarted, or stopped, you must supply at least 3 of these keys to unseal it
before it can start servicing requests.

Vault does not store the generated root key. Without at least 3 keys to
reconstruct the root key, Vault will remain permanently sealed!

It is possible to generate new unseal keys, provided you have a quorum of
existing unseal keys shares. See "vault operator rekey" for more information.
```
Copy and paste you're output in a text document.

You need at least 3 keys to unseal you're vault.

Paste 3 of 5 seal keys after:

```bash
kubectl exec -it vault-0 -n vault -- vault operator unseal <STRING OF UNSEAL KEY 1>
kubectl exec -it vault-0 -n vault -- vault operator unseal <STRING OF UNSEAL KEY 2>
kubectl exec -it vault-0 -n vault -- vault operator unseal <STRING OF UNSEAL KEY 3>
```

EXAMPLE INPUT:
```bash
kubectl exec -it vault-0 -n vault -- vault operator unseal GZsBfHCqzoculkpqyLWxplAh3J4oVZ1lPzPDJk7D6YK9
kubectl exec -it vault-0 -n vault -- vault operator unseal QRRB5Ihs6rI0WJDLjH3HV0iN/vxeA1LNbxnuPh6ZcqW5
kubectl exec -it vault-0 -n vault -- vault operator unseal RnacJwch9F9/1OtTlJeEbYSykLwReCkw7rNsOJoJUYoc
```

After unsealing login to the pod with root token:

```bash
kubectl exec -it vault-0 -n vault -- vault login <Initial Root Token>
```

EXAMPLE INPUT:
```bash
kubectl exec -it vault-0 -n vault -- vault login hvs.iZaGP4yIDxUSMqkaaqZ1XOJZ
```

##Enable and configure PKI engine in vault:

Enable PKI engine:

Ctrl+c en Ctrl+v
```bash
kubectl exec -it vault-0 -n vault -- vault secrets enable pki
```

##Renerate root CA:

Below will create private CA certificate for the k3s.internal domain wich is valid for 10 years

Ctrl+c en Ctrl+v
```bash
kubectl exec -it vault-0 -n vault -- vault write pki/root/generate/internal \
    common_name=k3s.internal \
    ttl=87600h
```

EXAMPLE OUTPUT:
```console
WARNING! The following warnings were returned from Vault:

  * This mount hasn't configured any authority information access (AIA)
  fields; this may make it harder for systems to find missing certificates
  in the chain or to validate revocation status of certificates. Consider
  updating /config/urls or the newly generated issuer with this information.

Key              Value
---              -----
certificate      -----BEGIN CERTIFICATE-----
MIIDODCCAiCgAwIBAgIURnZU5qSxP/GS4NBrY0OjqWiuNj4wDQYJKoZIhvcNAQEL
BQAwFzEVMBMGA1UEAxMMazNzLmludGVybmFsMB4XDTI0MDgyMTIxNTczMloXDTI0
MDkyMjIxNTgwMlowFzEVMBMGA1UEAxMMazNzLmludGVybmFsMIIBIjANBgkqhkiG
9w0BAQEFAAOCAQ8AMIIBCgKCAQEAuk2eUb7u2riW7KjLuEetJ0zqDnd6dkoOl26E
QVeq+EaJhPBKz0jVZtuqtECUm6G1NpXg/3pbVPG7N2ICmSMesV3ZiIXQGQRSBioP
0maWxLLPRx0hIHQKr4KZRXynFlWNK6/BrM2Yzw6usnKlUFiH9diIn8FGmb+nRcQS
jjvf0htWgWIlSxtn492JTYBtObp0nkKXuKgELp0+3iiwjNTX2yn0Tb4tY5O2lx4d
UQnUbFtakqqzz/oZkxj5Sfnuq0eay9ui8cIXRfCPxn9eLAYAkU/8iLvDVgXwyGmd
0EJQzpkoESI3kKHlEuIubMZsJ+tc2ctTYbR00OjJAoxCjqmYUQIDAQABo3wwejAO
BgNVHQ8BAf8EBAMCAQYwDwYDVR0TAQH/BAUwAwEB/zAdBgNVHQ4EFgQUzyd3yzvR
znJeZC9ZSPtOv5/Wqg8wHwYDVR0jBBgwFoAUzyd3yzvRznJeZC9ZSPtOv5/Wqg8w
FwYDVR0RBBAwDoIMazNzLmludGVybmFsMA0GCSqGSIb3DQEBCwUAA4IBAQCiLqYC
Rw9daaCKMCQwEp1/c9b0rJX3YW5H3Cq2vtv+MLWtjPXvoFx5VhbjWO1Za42i3Cq6
Dv0KzFW+tpLYlwhoRXNzbOcZ9gwskoz0j7Xm54sdDI/6XNFmQ24VZjdD9kf3E3vU
7FEySo87wZ1k3enYyeqDP+9BeFMJafhP5dtRg26GqWnASP0QnJiNhTAWIopBKXVY
6wgAJ2kVYZG9xiiOxiYShACvawhC/eD/8rK5eaEpvBAFeyIVjocZmy4p0fA7Aun5
GogDRoqiif4zK+7h0LaeuNkT5liKylr+Mn54ihJvftkRHSdes/yVwpkhYTCT3BwS
8EoV81eWLJUZbwXt
-----END CERTIFICATE-----
expiration       1727042282
issuer_id        e137683f-dbf3-7453-3c52-1312ecaf8078
issuer_name      n/a
issuing_ca       -----BEGIN CERTIFICATE-----
MIIDODCCAiCgAwIBAgIURnZU5qSxP/GS4NBrY0OjqWiuNj4wDQYJKoZIhvcNAQEL
BQAwFzEVMBMGA1UEAxMMazNzLmludGVybmFsMB4XDTI0MDgyMTIxNTczMloXDTI0
MDkyMjIxNTgwMlowFzEVMBMGA1UEAxMMazNzLmludGVybmFsMIIBIjANBgkqhkiG
9w0BAQEFAAOCAQ8AMIIBCgKCAQEAuk2eUb7u2riW7KjLuEetJ0zqDnd6dkoOl26E
QVeq+EaJhPBKz0jVZtuqtECUm6G1NpXg/3pbVPG7N2ICmSMesV3ZiIXQGQRSBioP
0maWxLLPRx0hIHQKr4KZRXynFlWNK6/BrM2Yzw6usnKlUFiH9diIn8FGmb+nRcQS
jjvf0htWgWIlSxtn492JTYBtObp0nkKXuKgELp0+3iiwjNTX2yn0Tb4tY5O2lx4d
UQnUbFtakqqzz/oZkxj5Sfnuq0eay9ui8cIXRfCPxn9eLAYAkU/8iLvDVgXwyGmd
0EJQzpkoESI3kKHlEuIubMZsJ+tc2ctTYbR00OjJAoxCjqmYUQIDAQABo3wwejAO
BgNVHQ8BAf8EBAMCAQYwDwYDVR0TAQH/BAUwAwEB/zAdBgNVHQ4EFgQUzyd3yzvR
znJeZC9ZSPtOv5/Wqg8wHwYDVR0jBBgwFoAUzyd3yzvRznJeZC9ZSPtOv5/Wqg8w
FwYDVR0RBBAwDoIMazNzLmludGVybmFsMA0GCSqGSIb3DQEBCwUAA4IBAQCiLqYC
Rw9daaCKMCQwEp1/c9b0rJX3YW5H3Cq2vtv+MLWtjPXvoFx5VhbjWO1Za42i3Cq6
Dv0KzFW+tpLYlwhoRXNzbOcZ9gwskoz0j7Xm54sdDI/6XNFmQ24VZjdD9kf3E3vU
7FEySo87wZ1k3enYyeqDP+9BeFMJafhP5dtRg26GqWnASP0QnJiNhTAWIopBKXVY
6wgAJ2kVYZG9xiiOxiYShACvawhC/eD/8rK5eaEpvBAFeyIVjocZmy4p0fA7Aun5
GogDRoqiif4zK+7h0LaeuNkT5liKylr+Mn54ihJvftkRHSdes/yVwpkhYTCT3BwS
8EoV81eWLJUZbwXt
-----END CERTIFICATE-----
key_id           c986886d-1493-bbd7-9212-eb657bb2ddf6
key_name         n/a
serial_number    46:76:54:e6:a4:b1:3f:f1:92:e0:d0:6b:63:43:a3:a9:68:ae:36:3e
```

Create a role to create certificates for subdomains of k3s.internal with a maximum lifetime of 7 days:

Ctrl+c en Ctrl+v
```bash
kubectl exec -it vault-0 -n vault -- vault write pki/roles/k3s \
    allowed_domains=k3s.internal \
    allow_subdomains=true \
    max_ttl=168h
```

Create a policy in vault, later we will assign it to a token:

Ctrl+c en Ctrl+v
```bash
kubectl exec -it vault-0 -n vault -- vault policy write pki-policy - <<EOF
path "pki/issue/k3s" {
  capabilities = ["create", "update"]
}
EOF
```

Create and assign token for this policy:

Ctrl+c en Ctrl+v
```bash
kubectl exec -it vault-0 -n vault -- vault token create -policy=pki-policy
```
OUTPUT:
```console
Key                  Value
---                  -----
token                hvs.CAESII12Eukd_dOjJ8tgPILPaV4vJDNjj-sk2n6R_ieVwxDzGh4KHGh2cy5CRWxIRWIwZEJBR0dKMXdXdTVDUHFsNVk
token_accessor       FPRc2bR4je2GNIU41ZqonlEJ
token_duration       768h
token_renewable      true
token_policies       ["default" "pki-policy"]
identity_policies    []
policies             ["default" "pki-policy"]
```

#Enable kubernetes authentication type kubernetes in vault

Ctrl+c en Ctrl+v
```bash
kubectl exec -it vault-0 -n vault -- vault auth enable kubernetes
```

Create k3s role with k3svault user account which is abled to fetch a login token that is usable for 24hours

Ctrl+c en Ctrl+v
```bash
kubectl exec -it vault-0 -n vault -- vault write auth/kubernetes/role/k3s \
  bound_service_account_names=k3svault \
  bound_service_account_namespaces=vault \
  policies=pki-policy \
  ttl=24h
```

##Apply the kubernetes service account with the cluster role and clusterrolebinding

This will create a service account in kubernetes named k3svault.
The clusterrole will give this service account priveliges.
The Clusterrolebinding will assign the service account to the service account

```bash
kubectl apply -f ./vault/vault-svc-account.yaml
kubectl apply -f ./vault/vault-clusterrole.yaml
kubectl apply -f ./vault/vault-clusterrolebinding.yaml
```

Below is optional because the above should automaticly create a secret for the service account user.

This will create a file called "vault-svc-acc-secret.yaml".

Ctrl+c and Ctrl+v:
```bash
cd ./vault
cat <<EOF > vault-svc-acc-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: k3s-vault-token
  namespace: vault
  annotations:
    kubernetes.io/service-account.name: k3svault
type: kubernetes.io/service-account-token
EOF
cd ../
```

This will apply the yaml above and create a secret in kubernetes:

Ctrl+c and Ctrl+v:
```bash
kubectl apply -f ./vault/vault-svc-acc-secret.yaml
```

##Create a certificate for vault.k3s.internal

Create directorys and cert files

Ctrl+c and Ctrl+v:
```bash
mkdir ./vault/certs
cd ./vault/certs/
touch certificate.pem private_key.pem
```

Create cert and store output in the file called cert_output.json
Ctrl+c and Ctrl+v:
```bash
kubectl exec -it vault-0 -n vault -- vault write pki/issue/k3s common_name="vault.k3s.internal" \
  alt_names="vault.k3s.internal" \
  ttl="168h" > cert_output.json
```

Show the content of the file to copy and paste the neccesary peaces in the corresponding files.

Ctrl+c and Ctrl+v:
```bash
cat cert_output.json
```

EXAMPLE OUTPUT:
```console
# cat cert_output.json
Key                 Value
---                 -----
ca_chain            [-----BEGIN CERTIFICATE-----
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
-----END CERTIFICATE-----]
certificate         -----BEGIN CERTIFICATE-----
MIIDVDCCAjygAwIBAgIUMFZqPK3cVthC+yjV2cSf5c6PqGowDQYJKoZIhvcNAQEL
BQAwFzEVMBMGA1UEAxMMazNzLmludGVybmFsMB4XDTI0MDgyOTE4MzMzNFoXDTI0
MDkwNTE4MzQwM1owHTEbMBkGA1UEAxMSdmF1bHQuazNzLmludGVybmFsMIIBIjAN
BgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAvg3GHdHgKSD0oY7Eq6C0G8LQot4J
Pte478HNP9rP4IeueacOfXfu5ZL47M+ZAgdNguETqLG+3J8M5OPOrRAL01dD+dUL
Up/miaHvRTacijX/4Mheq1ZR6YxOrhjIidcre3j1FWu3gvITlmm/EAm1AdG9oOa5
O8c5YdQf5318/Q3BV+KqGm/xJJ73g0IaKIfPDWoCwyM7WUrJ56SpkMAmPoVISWuQ
u6kl7z/facgaKoOQD0TVOEMadcAVXMCoWnpw8LeuqHyseH5W/XSIvt4zGYWz2P/u
gOULiiDLx8Pp6BsOl+MJqumqgwVy4/xs0CH4T0wrX3ufENsP4bZCWoMZkwIDAQAB
o4GRMIGOMA4GA1UdDwEB/wQEAwIDqDAdBgNVHSUEFjAUBggrBgEFBQcDAQYIKwYB
BQUHAwIwHQYDVR0OBBYEFEAbXndQXLbs/4MqELJGqKqRIWu3MB8GA1UdIwQYMBaA
FOnr1rRoWU+GsVk/QPGwEh0EW833MB0GA1UdEQQWMBSCEnZhdWx0Lmszcy5pbnRl
cm5hbDANBgkqhkiG9w0BAQsFAAOCAQEATVYeowbL9Kgd3PpgrzzbEptDOG/Y+s7Z
+hOqnF0X0SDhx+cbvAp0FDfQMQBbCN2rfXzQUlLCqzeo7dzYM6nvwGyqMD+W/VNr
tOYPMwN3ccfCgwjvjb4oAviaCKT/jF5XYf2YhLTWJif8JGHmUl4W+MQWYYF2oOC2
+B620RDMTJh6fq9/9i7KTnxXppfB/7gWj3mBhy/ZWJPEzLdZA+ovwE7CYF1s8SsF
9NE4Vp4JiTARSaMPg8Vq5kvu+I9E1dgygB+i3WdaW/IWHYDh9yBAYNeaKGkn7I+C
OSJHSW0+KOJFCAdk53rM5EEjDY04fwheoeEnkagIu178WWXD66jk3g==
-----END CERTIFICATE-----
expiration          1725561243
issuing_ca          -----BEGIN CERTIFICATE-----
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
private_key         -----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEAvg3GHdHgKSD0oY7Eq6C0G8LQot4JPte478HNP9rP4IeueacO
fXfu5ZL47M+ZAgdNguETqLG+3J8M5OPOrRAL01dD+dULUp/miaHvRTacijX/4Mhe
q1ZR6YxOrhjIidcre3j1FWu3gvITlmm/EAm1AdG9oOa5O8c5YdQf5318/Q3BV+Kq
Gm/xJJ73g0IaKIfPDWoCwyM7WUrJ56SpkMAmPoVISWuQu6kl7z/facgaKoOQD0TV
OEMadcAVXMCoWnpw8LeuqHyseH5W/XSIvt4zGYWz2P/ugOULiiDLx8Pp6BsOl+MJ
qumqgwVy4/xs0CH4T0wrX3ufENsP4bZCWoMZkwIDAQABAoIBAC5V2Vln08jzOfEx
h414brDd/FPY4lQp7/K0Q0AwLsJFEiqiqgu488uQ25OQwXMXKLSh/1L/ktLjDBe5
2qei498wxWfhoxMP3PrtOhKbz+p6Y9n/v+Tx9KKGDKCxdiL1DKrbwJTqYCFSt6fS
PDzCwRiidCMIXVzPo5PQTb74f0KKba9pmbL8APHytkO4G/KSbC1tj/NXOKuUE9Gf
dNz+hp2yrPhGQtgf5en/rb7lqJcEOq/1xOOAxlTvFDsDk46k4iHtNLX1eOD6o2Sv
mboeGgeslbMJSzXuR1jpOe12hDE1SzGNq5DKFXYsoen5xJcfFCgcMSLKJdxP/Vfi
8uhzd/ECgYEAxC0iJlZ3if9DsIEM8fBoQYSQxRlUDwBL0txeMRhzAoNsFSBM2Qes
kCM/sbIok40ZSW0+NWwJ5w+jB1qmdXi9n6dciT/SVRbeIBi0BdZOF1sWLfyEe9T1
r4LNtSyk5+BnMWXSvxEmH97lr8lN8Tjh+pj6IXvwqoDsMZrHnuo6ktcCgYEA+AKt
JGzQs5JgPaUU5OI9PuadGqgfy8EH8m87eMBINgZAy6zVWc7Lr28MIpa5M4u+6LVv
8H8RFZIOKukUOrqjAnYRb0ByeaEUsEmL89YThMm0jkt/vOXpL8G3Nmomk2sci6ly
KwemBmQhbVACC2wfu/iElY2csoMV97CkDyX3k6UCgYAxNT0GrtPHWq9o+8X6fho4
rP7/Ya4TITjjyIEcAYz/yWV4GyULn4Aqm5zjftPsxwzbvTpIfjQxsFttgdCVUNcH
0BxHFSo2S8kl9exaNnpaI2/50wiMY0vJXZ8p3evzefeIjYkCglO01N16bZ1Ob71H
dc3wTj19F1+nxbJi61AL+wKBgA/h8/6iLVdip2ErQkRKLMvrbuI3JBojWYPwFans
/nLfQaUJg3xF3wt0HB3W8zNW3rn+bJXFPW3ZNakP1ijQrQHKV+F9Che59h44B4tt
CUD2veZi9WI+gwl46WfFsoS8Vk6nYlVZHwvHu9BJUGg0229pQexl7kQMWwrKuCb0
Mn+1AoGBAKgihnPtm6Y9po04scEXyiBhctjgrrZjaKyUvP6VPdn9+zt3yaoyhc7E
MjJOqFESAAjHXfRZbrFuR9P3APRgUWELX2Q1Qm73vDuqwIpH1jGySZuYZEY8DgHm
6rS5e+mhmtdcv8MoTO3oArsM/Sac9SVW4TUJ4BAkVE9JLIkT/+CU
-----END RSA PRIVATE KEY-----
private_key_type    rsa
serial_number       30:56:6a:3c:ad:dc:56:d8:42:fb:28:d5:d9:c4:9f:e5:ce:8f:a8:6a
```

Search for private_key copy and paste in private_key.pem.

Example copy part:
```console
-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEAvg3GHdHgKSD0oY7Eq6C0G8LQot4JPte478HNP9rP4IeueacO
fXfu5ZL47M+ZAgdNguETqLG+3J8M5OPOrRAL01dD+dULUp/miaHvRTacijX/4Mhe
q1ZR6YxOrhjIidcre3j1FWu3gvITlmm/EAm1AdG9oOa5O8c5YdQf5318/Q3BV+Kq
Gm/xJJ73g0IaKIfPDWoCwyM7WUrJ56SpkMAmPoVISWuQu6kl7z/facgaKoOQD0TV
OEMadcAVXMCoWnpw8LeuqHyseH5W/XSIvt4zGYWz2P/ugOULiiDLx8Pp6BsOl+MJ
qumqgwVy4/xs0CH4T0wrX3ufENsP4bZCWoMZkwIDAQABAoIBAC5V2Vln08jzOfEx
h414brDd/FPY4lQp7/K0Q0AwLsJFEiqiqgu488uQ25OQwXMXKLSh/1L/ktLjDBe5
2qei498wxWfhoxMP3PrtOhKbz+p6Y9n/v+Tx9KKGDKCxdiL1DKrbwJTqYCFSt6fS
PDzCwRiidCMIXVzPo5PQTb74f0KKba9pmbL8APHytkO4G/KSbC1tj/NXOKuUE9Gf
dNz+hp2yrPhGQtgf5en/rb7lqJcEOq/1xOOAxlTvFDsDk46k4iHtNLX1eOD6o2Sv
mboeGgeslbMJSzXuR1jpOe12hDE1SzGNq5DKFXYsoen5xJcfFCgcMSLKJdxP/Vfi
8uhzd/ECgYEAxC0iJlZ3if9DsIEM8fBoQYSQxRlUDwBL0txeMRhzAoNsFSBM2Qes
kCM/sbIok40ZSW0+NWwJ5w+jB1qmdXi9n6dciT/SVRbeIBi0BdZOF1sWLfyEe9T1
r4LNtSyk5+BnMWXSvxEmH97lr8lN8Tjh+pj6IXvwqoDsMZrHnuo6ktcCgYEA+AKt
JGzQs5JgPaUU5OI9PuadGqgfy8EH8m87eMBINgZAy6zVWc7Lr28MIpa5M4u+6LVv
8H8RFZIOKukUOrqjAnYRb0ByeaEUsEmL89YThMm0jkt/vOXpL8G3Nmomk2sci6ly
KwemBmQhbVACC2wfu/iElY2csoMV97CkDyX3k6UCgYAxNT0GrtPHWq9o+8X6fho4
rP7/Ya4TITjjyIEcAYz/yWV4GyULn4Aqm5zjftPsxwzbvTpIfjQxsFttgdCVUNcH
0BxHFSo2S8kl9exaNnpaI2/50wiMY0vJXZ8p3evzefeIjYkCglO01N16bZ1Ob71H
dc3wTj19F1+nxbJi61AL+wKBgA/h8/6iLVdip2ErQkRKLMvrbuI3JBojWYPwFans
/nLfQaUJg3xF3wt0HB3W8zNW3rn+bJXFPW3ZNakP1ijQrQHKV+F9Che59h44B4tt
CUD2veZi9WI+gwl46WfFsoS8Vk6nYlVZHwvHu9BJUGg0229pQexl7kQMWwrKuCb0
Mn+1AoGBAKgihnPtm6Y9po04scEXyiBhctjgrrZjaKyUvP6VPdn9+zt3yaoyhc7E
MjJOqFESAAjHXfRZbrFuR9P3APRgUWELX2Q1Qm73vDuqwIpH1jGySZuYZEY8DgHm
6rS5e+mhmtdcv8MoTO3oArsM/Sac9SVW4TUJ4BAkVE9JLIkT/+CU
-----END RSA PRIVATE KEY-----
```

Copy and paste the certificate part into certificate.pem.

Example copy part:
```console
-----BEGIN CERTIFICATE-----
MIIDVDCCAjygAwIBAgIUMFZqPK3cVthC+yjV2cSf5c6PqGowDQYJKoZIhvcNAQEL
BQAwFzEVMBMGA1UEAxMMazNzLmludGVybmFsMB4XDTI0MDgyOTE4MzMzNFoXDTI0
MDkwNTE4MzQwM1owHTEbMBkGA1UEAxMSdmF1bHQuazNzLmludGVybmFsMIIBIjAN
BgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAvg3GHdHgKSD0oY7Eq6C0G8LQot4J
Pte478HNP9rP4IeueacOfXfu5ZL47M+ZAgdNguETqLG+3J8M5OPOrRAL01dD+dUL
Up/miaHvRTacijX/4Mheq1ZR6YxOrhjIidcre3j1FWu3gvITlmm/EAm1AdG9oOa5
O8c5YdQf5318/Q3BV+KqGm/xJJ73g0IaKIfPDWoCwyM7WUrJ56SpkMAmPoVISWuQ
u6kl7z/facgaKoOQD0TVOEMadcAVXMCoWnpw8LeuqHyseH5W/XSIvt4zGYWz2P/u
gOULiiDLx8Pp6BsOl+MJqumqgwVy4/xs0CH4T0wrX3ufENsP4bZCWoMZkwIDAQAB
o4GRMIGOMA4GA1UdDwEB/wQEAwIDqDAdBgNVHSUEFjAUBggrBgEFBQcDAQYIKwYB
BQUHAwIwHQYDVR0OBBYEFEAbXndQXLbs/4MqELJGqKqRIWu3MB8GA1UdIwQYMBaA
FOnr1rRoWU+GsVk/QPGwEh0EW833MB0GA1UdEQQWMBSCEnZhdWx0Lmszcy5pbnRl
cm5hbDANBgkqhkiG9w0BAQsFAAOCAQEATVYeowbL9Kgd3PpgrzzbEptDOG/Y+s7Z
+hOqnF0X0SDhx+cbvAp0FDfQMQBbCN2rfXzQUlLCqzeo7dzYM6nvwGyqMD+W/VNr
tOYPMwN3ccfCgwjvjb4oAviaCKT/jF5XYf2YhLTWJif8JGHmUl4W+MQWYYF2oOC2
+B620RDMTJh6fq9/9i7KTnxXppfB/7gWj3mBhy/ZWJPEzLdZA+ovwE7CYF1s8SsF
9NE4Vp4JiTARSaMPg8Vq5kvu+I9E1dgygB+i3WdaW/IWHYDh9yBAYNeaKGkn7I+C
OSJHSW0+KOJFCAdk53rM5EEjDY04fwheoeEnkagIu178WWXD66jk3g==
-----END CERTIFICATE-----
```

When files are ready create a kubernetes secret that will contain the cert and key you created.

Ctrl+c and Ctrl+v:
```bash
kubectl create secret tls vault-k3s-internal \
 --cert=certificate.pem \
 --key=private_key.pem \
 -n vault
```

Now we will apply the ingress for the vault service by doing:

Ctrl+c and Ctrl+v:
```bash
kubectl apply -f ./vault/vault-ingress.yaml
```
Make sure that you have created a DNS entry for *.k3s.internal pointing to the api server ip address.
All traffic will be routed trough that ip, and when the fqdn is used corresponding one of the values in the ingress attribute known by kubernetes it will be connected to the corresponding service.