#Установка кибернетиса [Инструкция которой руководствовался я](https://ex-minds.ru/?p=296)


##setting

```bash
sudo -s

printf "overlay\nbr_netfilter\n" >> /etc/modules-load.d/containerd.conf

modprobe overlay
modprobe br_netfilter

printf "net.bridge.bridge-nf-call-iptables = 1\nnet.ipv4.ip_forward = 1\nnet.bridge.bridge-nf-call-ip6tables = 1\n" >> /etc/sysctl.d/99-kubernetes-cri.conf

sysctl --system

wget https://github.com/containerd/containerd/releases/download/v1.6.16/containerd-1.6.16-linux-amd64.tar.gz -P /tmp/
tar Cxzvf /usr/local /tmp/containerd-1.6.16-linux-amd64.tar.gz
wget https://raw.githubusercontent.com/containerd/containerd/main/containerd.service -P /etc/systemd/system/
systemctl daemon-reload
systemctl enable --now containerd

wget https://github.com/opencontainers/runc/releases/download/v1.1.4/runc.amd64 -P /tmp/
install -m 755 /tmp/runc.amd64 /usr/local/sbin/runc

wget https://github.com/containernetworking/plugins/releases/download/v1.2.0/cni-plugins-linux-amd64-v1.2.0.tgz -P /tmp/
mkdir -p /opt/cni/bin
tar Cxzvf /opt/cni/bin /tmp/cni-plugins-linux-amd64-v1.2.0.tgz

mkdir -p /etc/containerd
containerd config default | tee /etc/containerd/config.toml   <<<<<<<<<<< manually edit and change systemdCgroup to true
systemctl restart containerd

swapoff -a  <<<<<<<< just disable it in /etc/fstab instead

apt-get update
```

##docker

```bash
sudo apt-get remove docker docker-engine docker.io containerd runc
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg lsb-release
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io

```
##kubernetes

```bash
apt-get install -y apt-transport-https ca-certificates curl

sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://dl.k8s.io/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update -y

sudo apt-get install kubeadm kubelet kubectl kubernetes-cni
sudo apt-mark hold kubeadm kubectl kubelet kubernetes-cni
```
`На этой стадии можно создать резервную копию нашего контейнера или выполнить операцию ‘Convert to template’. Я выбрал второй вариант, чтобы при разворачивании последующих контейнеров не устанавливать docker и kubernetes. Единственное что нужно помнить при таком подходе — это что после создания клона из шаблона необходимо менять IP-адрес, ну и возможно параметры выделяемых ресурсов, таких как диск, память, процессор.
`



# Мастер нода

```bash
sudo swapoff -a
```

Инициализируем кластер 

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

выполняем:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Добавляем сеть [flannel](https://github.com/flannel-io/flannel#flannel):

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

`Мастер нода готова!`

# Подключение worker-нод

Если делали клон то:

меняем сеть:
```bash
sudo nano /etc/netplan/00-installer-config.yaml
sudo netplan apply
```


меняем название машины (Повторятся не должны!):
```bash
sudo hostnamectl set-hostname kubernetesworker01
```
и меняем ID:
```bash
sudo rm /etc/machine-id
sudo systemd-machine-id-setup
sudo reboot now
```

`Воркер-ноды системно ничем не отличаются от мастер-ноды, и по этой причине я просто клонировал два контейнера из созданного мной ранее шаблона контейнера, поменял IP-адреса, добавил им дискового пространства — до 50Гб и урезал колличество потоков CPU до двух. 
Думаю, пока хватит.`
После успешного запуска master-ноды в конце вывода должна быть строка для подключения worker-нод

```
kubeadm join <IP_мастер-ноды>:6443 --token g7mku9.52m1do9u3t58mfeb \
    --discovery-token-ca-cert-hash sha256:1be8897143e616cb2ce270865a12eb9e09242eea8da461a250f481554cc9fe1b 
```
Ошибка на которую наткнулся я забыв переименовать ноду:

[Kubernetes Master Worker Node Kubeadm Join issue](https://stackoverflow.com/questions/55767652/kubernetes-master-worker-node-kubeadm-join-issue)

Решение:
```bash
kubeadm reset 
```

Или так, что приведет к тому же результату, но будет создан новый токен, который будет действителен в течении суток:
```bash
kubeadm token create --print-join-command
```
В итоге на местер-ноде, выполнив команду ```kubectl get nodes```, мы должны увидеть следующую картину:
```
NAME           STATUS   ROLES                  AGE     VERSION
kube-master    Ready    control-plane,master   56m     v1.23.0
kube-worker1   Ready    <none>                 9m49s   v1.23.0
kube-worker2   Ready    <none>                 52s     v1.23.0
```


## Полезное

[Инструкция 1](https://ex-minds.ru/kubernetes-proxmox-install/)

[Инструкция 2](https://www.itsgeekhead.com/tuts/kubernetes-126-ubuntu-2204.txt)

[Еще инструкция мало ли может что упускаю](https://phoenixnap.com/kb/install-kubernetes-on-ubuntu)

[kubectl - Шпаргалка](https://kubernetes.io/ru/docs/reference/kubectl/cheatsheet/)

[неизвестная служба runtime.v1alpha2.ImageService](https://github.com/kubernetes-sigs/cri-tools/issues/710)

[Как отключить swap на Ubuntu, Debian](https://disnetern.ru/how-disable-swap-linux/)

[Как добавить роли узлам в Kubernetes?](https://stackoverflow.com/questions/48854905/how-to-add-roles-to-nodes-in-kubernetes)

Добавить роль
```bash
kubectl label node <node name> node-role.kubernetes.io/<role name>=<key - (any name)>
```
Удалить роль
```bash
kubectl label node <node name> node-role.kubernetes.io/<role name>-
```
