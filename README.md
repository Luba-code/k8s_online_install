# Kubernetes 前置作業與安裝

---

* 前置作業

![](https://i.imgur.com/sX9gZO5.jpg)


由於是用Ubuntu來做K8S的佈置，在VMware的開啟清單要先分類好M1和W1

![](https://i.imgur.com/oIaeQkA.jpg)

再來是設定好IP遮罩等等

```
# This is the network config written by 'subiquity'
network:
  ethernets:
    ens32:
      dhcp4: false
      addresses:
      - 192.168.218.21/24
      gateway4: 192.168.218.2
      nameservers:
        addresses: [8.8.8.8]
  version: 2
```

```
sudo netplan try
sudo netplan apply
```

![](https://i.imgur.com/9HasiFN.jpg)

永久關閉Ubuntu的IPv6

```
sudo nano /etc/default/grub
GRUB_CMDLINE_LINUX="ipv6.disable=1"
sudo update-grub
reboot
```

![](https://i.imgur.com/oxYNUnu.jpg)

把主機名稱改好

![](https://i.imgur.com/VpepF0H.jpg)

去增加M1和W1的名稱解析

##  以上步驟W1也做一次，做完就可以正式來佈置K8S

---

* 正式佈置K8S

```
sudo swapoff -a
sudo nano /etc/fstab
#/swap.img       none    swap    sw      0       0   加上註解
```

關閉swap,swap就是當電腦實體記憶體不夠用時，會使用硬碟局部空間作為記憶體的擴充，一旦使用了swap 電腦速度會變非常慢(虛擬記憶體),k8s 是叢集 記憶體不夠 就會自動把pod換到能執行的主機上 根本不用swap

![](https://i.imgur.com/iu6rlCA.png)

```
sudo nano /etc/sysctl.conf
net.ipv4.ip_forward = 1  把它前面的(#)註解拿掉
sudo sysctl -p
```
讓Ubuntu有route功能

![](https://i.imgur.com/B333HIx.jpg)

---

* 安裝cri-o  (這邊不是用docker)

```
sudo nano /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/xUbuntu_20.04/ /
```
```
sudo nano /etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:1.20.list
deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/1.20/xUbuntu_20.04/ /
```
上述安裝的是1.20版

```
curl -L https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:1.20/xUbuntu_20.04/Release.key |sudo  apt-key add -
curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/xUbuntu_20.04/Release.key |sudo  apt-key add -
```
```
sudo apt-get update
sudo apt-get install cri-o cri-o-runc -y
```
正式下載cri-o

`sudo sed -i 's/10.85.0.0\/16/10.244.0.0\/16/g' /etc/cni/net.d/100-crio-bridge.conf`

修改IP位置設定自己設定的  10.244.0.0是pod用的位置

```
sudo systemctl daemon-reload
sudo systemctl status crio
sudo systemctl start crio
sudo systemctl enable crio
```
針對k8s 需要的daemon-reload
enable 類似 rc 下次開機自動帶起來

![](https://i.imgur.com/wjYkdzU.jpg)

```
crio version
sudo reboot
```
查看 crio 版本

![](https://i.imgur.com/vFAOLzL.jpg)

---

* 安裝K8S

```
sudo nano /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
```
```
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
```
拿到對方網站的key 才能做安裝動作

```
sudo apt update
sudo apt-cache madison kubeadm | grep 1.20.2
```

![](https://i.imgur.com/09HNaer.jpg)

```
export K_VER=1.20.2-00
sudo apt install -y kubelet=${K_VER} kubectl=${K_VER} kubeadm=${K_VER}
sudo apt-mark hold kubelet kubeadm kubectl
```
宣告我k8s 鎖定在1.20.2 不會因為更新改變

```
sudo nano /etc/default/kubelet
KUBELET_EXTRA_ARGS=--feature-gates="AllAlpha=false,RunAsGroup=true" --container-runtime=remote --cgroup-driver=systemd --container-runtime-endpoint='unix:///var/run/crio/crio.sock' --runtime-request-timeout=5m
```
```
sudo systemctl daemon-reload
sudo systemctl enable kubelet
```
```
sudo kubeadm config images pull --kubernetes-version=v1.20.2
sudo nano /etc/modules
br_netfilter
sudo reboot
```
所有的k8s 1.20.2 image版本下載，啟動網路過濾器這個模組

![](https://i.imgur.com/51354VV.jpg)

## 到這邊不管是Master機和Worker機都要做到這裡

---

* K8S的M1和W1佈置

# 以下是針對M1

`sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --cri-socket=/var/run/crio/crio.sock`

啟動K8S，它會產生出一行憑證是給worker機做加入此K8S用的，每次產生的都不一樣

```
kubeadm join 192.168.218.21:6443 --token 560k2y.2eeoczxs7ssri82n \
    --discovery-token-ca-cert-hash sha256:755d5f394ebcfda676ac193a84f008bc762ed70b5cf20790410ea792c01ba4e3
```
![](https://i.imgur.com/qVzFmVx.jpg)

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
把bigred變成這m1的管理者帳號,一定要建 .kube資料夾,把k8s的admin設定檔複製給bigred,把設定檔更改擁有者和群組改成bigred,這樣bigred就是完整的K8S管理者

```
sudo apt install ipset
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

佈置K8S的網路平台flannel

`kubectl get pod -n kube-system`

## 查看你的K8S有沒有裝好，上面那段非常重要，因為如果有哪個環節出錯都優先來這裡看就知道了

![](https://i.imgur.com/x4VneIU.jpg)

`kubectl taint node m1-ubuntu node-role.kubernetes.io/master:NoSchedule-`

看個人要不要讓M1跑pod，通常M1是發送命令給其他Worker機工作的，+了上面那行的話你的Master機器就可以跑pod，通常視情況再做此動作，可以上網查minikube，(M1-Ubuntu是我這台M1的電腦名稱)，但這有規定節點都小寫所以要改成m1-ubuntu

`sudo kubeadm token create --print-join-command `

假如上面給worker機的憑證沒了的話，在Master機器上打上面那行就會產生了

![](https://i.imgur.com/VL5yeRy.jpg)

---

# 以下是針對W1

 `sudo kubeadm join 192.168.218.21:6443 --token mdemff.4ybrxejwz7u4jt4g     --discovery-token-ca-cert-hash sha256:755d5f394ebcfda676ac193a84f008bc762ed70b5cf20790410ea792c01ba4e3 `

把剛剛在M1產生的這串直接貼在Worker機器上，就能變能這套K8S的叢集電腦了

![](https://i.imgur.com/NqP6AIs.jpg)

---

# 回到M1,確認W1有加入叢集裡面

`kubectk get nodes`

可以看這加入此K8S的節點(worker)，發現worker機沒貼上標籤

`kubectl label node w1-ubuntu node-role.kubernetes.io/worker=worker`

補上標籤，完成K8S佈置(w1-ubuntu)是我的W1電腦名稱一樣要小寫

![](https://i.imgur.com/lzfhvdA.jpg)

#### 以上K8S佈置完成，由於我電腦配備不足，在此就佈置M1和W1兩台

###### tags: `Kubernetes`
