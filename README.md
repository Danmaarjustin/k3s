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

























