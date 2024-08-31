#Install master node

Login and swtich to root

Ctrl+c and Ctrl+v:
```bash
su
echo set nocompatible > /root/.vimrc
apt install sudo curl git gpg ca-certificates apt-transport-https gnupg nfs-common -y
hostnamectl set-hostname k-master2
echo k-master2 > /etc/hostname
vi /etc/hosts
```

Editor will be opend and edit the current hostname to new hostname.

Reboot the system!

##Instal k3s:

#On first master node When no Proxy in front of cluster:

Ctrl+c and Ctrl+v:
```bash
curl -sfL https://get.k3s.io | sh -s - server \
--token=<MAAK-HIER-IETS-LEUKS-VAN>
--cluster-init --write-kubeconfig-mode 644
```

When you are using a proxy/loadbalncer in front do, kan je ook doen door NAT regel te maken op je router/fireall, aangezien verkeer daar toch altijd al doorheen gaat:

Copy and paste to all master nodes do:

Ctrl+c and Ctrl+v:
```bash
curl -sfL https://get.k3s.io | sh -s - server \
--token=<MAAK-HIER-IETS-LEUKS-VAN>
--tls-san <VIP-ADDRESS> --tls-san <VIP-IP-ADDRESS>
--cluster-init
```
Get token:
Ctrl+c and Ctrl+v:
```bash
cat /var/lib/rancher/k3s/server/node-token
```
Copy token and past in text document.

EXAMPLE OUTPUT:
```console
K10acf961dac4ff6d4968497aced3f225d4663aad146bf313db746c0a816ed59418::server:391bad00f60f3fb2998e1739b3ec968f
```

On the  2 other master nodes:
Ctrl+c and Ctrl+v:
```bash
curl -sfL https://get.k3s.io | K3S_URL=https://<master1-ip>:6443 K3S_TOKEN=<your-token> sh -
```

When its finished installing its automaticly started.

#How kubernetes wokrs:

The kubernetes API can only be accessed from this moment when you're kubectl config or system has:
```console
/etc/rancher/k3s/k3s.yaml
```
This file contains root priviliges and certificates to authorizate and authanticate against the API server.
In the example below i will do everything on the first master node. But for example;

Everything can be applied and created in you're k3s cluster when you have the k3s.yaml.
If youre system environment has:
```console
export KUBECONFIG=k3s.yaml
```
or and export of a copy of it kubectl knows how to connect to you're k3s API.

#make you're life easyer:

Ctrl+c and Ctrl+v:
```bash
echo "alias k=$(which kubectl)" >> ~/.bashrc
```
or
Ctrl+c and Ctrl+v:
```bash
echo "alias k=/usr/local/bin/kubectl" >> ~/.bashrc
```
#Make a .kube dir in home folder and copy kubernetes config file in it.
This is for authenticating to you're kubernetes API.

Ctrl+c and Ctrl+v:
```bash
mkdir -p ~/.kube
cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
```

If you want the system user to be able to access the kubernetes api:

Ctrl+c and Ctrl+v:
```bash
mkdir ~/.kube/ ~/.kube/config
cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
chown $(whoami):$(whoami) ~/.kube/config
echo "export KUBECONFIG=~/.kube/config" >> ~/.bashrc
source ~/.bashrc
```
#Create directorys in home folder of user(No need when you have cloned this repo):

Ctrl+c and Ctrl+v:
```bash
mkdir k3s
mkdir k3s/rancher
mkdir k3s/awx
mkdir k3s/gitea
mkdir k3s/vault
mkdir k3s/zabbix
mkdir k3s/traefik
mkdir k3s/argocd
```


#Install helm

Ctrl+c and Ctrl+v:
```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```
or # Only for Debian and Ubuntu 
Ctrl+c and Ctrl+v:
```bash
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

#Create serviceaccount and clusterrolebinding for helm in you're k3s cluster:

Ctrl+c and Ctrl+v:
```bash
kubectl create serviceaccount --namespace kube-system
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin 
```

Innitialize helm:

Ctrl+c and Ctrl+v:
```bash
helm repo add stable https://charts.helm.sh/stable
helm repo update
```



##Mount nfs synology see ----> https://youtu.be/uPt3VKQOMBs?si=Qk2-f3jG61OQjdfr

We use /mnt #for example
Ctrl+c and Ctrl+v:
```bash
mkdir /mnt
mount -t nfs <IP-ADDRESS-SYNOLOGY>:volume1/<NAME-OF-SHARE> /mnt
```

Consideration doing(Discuss this maybe a other option for synology):
```bash
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm install nfs-client nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --set nfs.server=<nfs-server-ip> \
  --set nfs.path=/srv/nfs/kubedata
```	

When we did do this, ensure the NFS storage class is set as the default:


```bash
kubectl patch storageclass nfs-client -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```



#Install worker nodes

```bash
su
apt install sudo curl git gpg ca-certificates apt-transport-https gnupg nfs-common -y
hostnamectl set-hostname k-worker1
echo k-worker1 > /etc/hostname
/usr/sbin/reboot now
```
When its up and running:

Ctrl+c and Ctrl+v:
```bash
curl -sfL https://get.k3s.io | K3S_URL=https://<MASTER-IP>:6443 K3S_TOKEN=<MASTER-TOKEN> sh -
```
Repeat steps for more than 1 worker


#Install kubectl on debian:

```bash
sudo apt update
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg 
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo chmod 644 /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubectl
```


