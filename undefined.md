---
description: 설치 및 실행 방법
---

# 설치

#### [Index](https://gitlab.handymes.com/README.md) > AP Install > K-PaaS AP - min

### Table of Contents

1. 개요\
   1.1. 목적\
   1.2. 범위\
   1.3. 참고 자료
2. uengine-dev-platform 설치\
   2.1. Prerequisite\
   2.2. 가상화 구성을 위한 proxmox 설치\
   2.3. kubernetes vm template 생성 및 설치\
   2.3.1. vm 필수 package 설치\
   2.3.2. kubernetes cluster 설정 전 vm 요구 사항\
   2.3.3. install kubernetes cluster with kubekey\
   2.4. Gitpod 설치

## 1. 문서 개요

### 1.1. 목적

본 문서는 uengine-dev-platform(이하 udp)을 수동으로 설치하기 위한 가이드를 제공하는 데 그 목적이 있다.

\


### 1.2. 범위

udp는 kubekey를 기반으로 한 kubernetes 환경에서 설치하며 kubernetes dashboard는 kubesphere를 사용하였다. udp는 xenserver, proxmox, VMware vSphere, Google Cloud Platform, Amazon Web Services EC2, OpenStack, Microsoft Azure 등의 IaaS를 지원하며, 본문서의 적용 환경은 proxmox, kvm 환경이다.

\


### 1.3. 참고 자료

본 문서는 다양한 개발사/개발자의 Document를 참고로 작성하였다

Proxmox (가상화): [https://pve.proxmox.com/pve-docs/](https://pve.proxmox.com/pve-docs/)\
Kind (경량화 k8s cluster 생성): [https://kind.sigs.k8s.io/](https://kind.sigs.k8s.io/)\
Gitpod (cloud ide 배포 및 관리): [https://www.gitpod.io/](https://www.gitpod.io/)\
kubesphere (k8s 배포 및 관리): [https://kubesphere.io/](https://kubesphere.io/)

\
\


## 2. uengine-dev-platform 설치

### 2.1. Prerequisite

* CPU는 18core, memory는 128G, 1gb network, Disk(sas 또는 ssd)는 2TB 이상의 spec을 가진 baremetal 장비
* cli로 접근 가능한 dns에 domain (amazon, cloudflare 등)
* IaaS설치를 위한 proxmox 설치 image를 download 하고 usb 도는 cd로 설치 가능한 disk를 만든다. (https://www.proxmox.com/en/downloads/proxmox-virtual-environment/iso)

\


### 2.2. 가상화 구성을 위한 proxmox 설치

* 일반 linux 설치 절차와 유사하며 자세한 설치는 https://www.linuxtechi.com/install-proxmox-ve-on-bare-metal/ 를 참조한다.
* Table of Contents

```
   Prerequisites
1) Download Proxmox VE ISO Installer
2) Boot from the Proxmox VE Installer
3) Accept End User License Agreement
4) Choose Installation Target Disk
5) Set Location and Time Zone
6) Create Admin Password and Email Address
7) Network Configuration
8) Begin Proxmox VE Installation
9) Access the Proxmox VE Web Interface
10) Change Proxmox Enterprise Repository to Community
```

* proxmox 서버에 TLS 설정 (도메인 이름이 handymes.com 일경우)

```bash
$ git clone  https://github.com/acmesh-official/acme.sh.git acme.sh-master
$ cd acme.sh-master/
$ sudo ./acme.sh --register-account -m zasmin@uengine.net
$ sudo ./acme.sh --issue --standalone --keypath /etc/pve/local/pve-ssl.key --server letsencrypt --fullchainpath /etc/pve/local/pve-ssl.pem --reloadcmd "systemctl restart pveproxy" -d proxmox3.handymes.com
```

* proxmox public licenses 관련 팝업 제거

```bash
$ cd /usr/share/javascript/proxmox-widget-toolkit/
$ cp proxmoxlib.js proxmoxlib.js.bak
$ vi proxmoxlib.js
    Ext.Msg.show({      ----->     void ({ // Ext.Msg.show({
$ sudo systemctl restart pveproxy.service
```

\


### 2.3. kubernetes vm template 생성 및 설치

* 본 예에서는 ubuntu 22.04 를 base로 표준 image를 생성
* ubuntu image repo: https://cloud-images.ubuntu.com/
* create proxmox vm template: https://4sysops.com/archives/how-to-create-a-proxmox-vm-template/

```bash
# proxmox 서버 cli 
$ wget https://cloud-images.ubuntu.com/jammy/20230613/jammy-server-cloudimg-amd64.img
$ mv jammy-server-cloudimg-amd64.img jammy.qcow2 # qcow2로 변경
$ qemu-img info jellyfish.qcow2
$ qemu-img resize jellyfish.qcow2 32G  # disk size 32G로 변경
# 아래 a,b 작업 완료 후
$ qmset 100 --serial0 socket --vga serial0
$ qm importdisk 900 jammy.qcow2 proxmox
```

```
a. proxmox web gui로 vm 생성
b. CreateVM > General (node,vmid,name) > OS (Do not use any  media) >  System (Qemu Agent) > Hard Disk ( scsi device 제거) > CPU > Memory > Network > Confirm > 확인
```

```
proxmox web gui에서 방금 생성한 disk edit > add
900 > options > Boot Order scsi0 disk 1번 또는 2번으로 변경후 start
```

#### 2.3.1 vm 필수 package 설치

* ubuntu os daemon 관리 popup 창 제거

```bash
$ vi /etc/needrestart/needrestart.conf
...
    $nrconf{restart} = 'l';  # i -> l
...
```

* containerd 설치 (kubernetes 1.21 이후 버전부터 docker deprecated )

```bash
# containerd v1.6.3 has a bug and doesn’t start containers due to issues with CNI on Kubernetes v1.23 
wget https://github.com/containerd/containerd/releases/download/v1.6.2/containerd-1.6.2-linux-amd64.tar.gz
sudo tar Czxvf /usr/local containerd-1.6.2-linux-amd64.tar.gz

wget https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
sudo mv containerd.service /usr/lib/systemd/system/

sudo systemctl daemon-reload
sudo systemctl enable --now containerd
sudo systemctl status containerd
```

* install runC

> runC is an open-source container runtime for spawning and running containers on Linux according to the OCI specification.

```bash
wget https://github.com/opencontainers/runc/releases/download/v1.1.1/runc.amd64
sudo install -m 755 runc.amd64 /usr/local/sbin/runc
```

* containerd configuration for k8s

```bash
sudo mkdir -p /etc/containerd/
$ containerd config default | sudo tee /etc/containerd/config.toml

$ sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
$ sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml

$ sudo systemctl restart containerd
```

* 기타 주요 os package 설치

```bash
$ sudo apt install -y nfs-common cifs-utils socat conntrack ebtables ipset ipvsadm jq
$ sudo wget -qO /usr/local/bin/yq https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64
$ sudo chmod +x /usr/local/bin/yq
```

* local time 설정

```bash
$ sudo rm /etc/localtime
$ sudo ln -s /usr/share/zoneinfo/ROK /etc/localtime
```

* aws client 설치

```bash
$ curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
$ unzip awscliv2.zip
$ sudo ./aws/install --update
```

* minio client 설치 (https://min.io/docs/minio/kubernetes/upstream/)

> MinIO는 오픈소스로 제공되는 오브젝트 스토리지 서버이며 AWS S3와의 호환되는 클라우드 스토리지를 구축할 수 있다.

```bash
$ wget https://dl.min.io/client/mc/release/linux-amd64/mc
$ chmod +x mc && mv mc /usr/local/bin
$ mc config host list
```

* kubekey 설치 (https://kubesphere.io/docs/v3.4/installing-on-linux/introduction/kubekey/)

> Go 로 개발된 kubekey는 Ansible 기반 설치 프로그램을 대체하는 새로운 설치 도구이며 kubernetes 또는 kubesphere를 모두 설치할 수 있다.

```bash
$ curl -sfL https://get-kk.kubesphere.io | VERSION=v3.0.7 sh -
$ sudo mv kk /usr/local/bin
```

* delete contents in /etc/machine-id

```bash
# machine id 를 삭제하여 생성된 vm template로 vm 을 cloning하여 생성할 경우 새로운 machine-id를 적용한다.
$ sudo echo "" > /etc/machine-id
```

#### 2.3.2 kubernetes cluster 설정 전 vm 요구 사항

* vm은 master, worker (node1, node2, node3) 3개로 구성한다.

> node spec

```
master node:
    cloud-init에서 고정 ip 할당
    4vcpu/8GB
worker node:
    cloud-init에서 고정 ip 할당
    8vcpu/32GB
    proxmox gui에서 필요한 용량의 disk를 각 node에 추가
      - /var/lib/containerd (gitpod overlay filesystem 저장소)
            gitpod 구성시 workspace가 구성
      - /var/openebs/local (storageClass 구성)
            hostPath로 CSI storage 저장소
```

* master node ssh key 생성 및 등록 (root 계정사용)

```bash
# RSA 보다 간결한 암호화 기능을 제공하는 ed25519 를 사용
ssh-keygen -t ed25519
```

```bash
# master server
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authrized_keys
# 각 node의 root 계정의 .ssh/authrized_keys에 등록 (아래 key는 sample임)
echo "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIHtYJGYZHsYMgRAFNYERH1PXxZ1Oj/gJ0YhLXE0s3ZAH root@master" >> ~/.ssh/authrized_keys
```

* vm disk 구성 (root 계정사용)

> proxmox gui에서 각 worker node에 2개의 disk를 할당한 상태

```bash
# vm에 할당된 disk 확인
lsblk
# 할당된 disk filesystem 및 mount point 생성 (storageClass)
vgcreate -s 4m scdata /dev/sdb # storageClass 사용 
lvcreate --extents 100%FREE -n data scdata
mkfs.ext4 /dev/pvdata/data
mkdir -p /var/openebs/local
# 할당된 disk filesystem 및 mount point 생성 (gitpodData)
vgcreate -s 4m gitdata /dev/sdb # storageClass 사용 
lvcreate --extents 100%FREE -n data scdata
mkfs.ext4 /dev/gitdata/data
mkdir -p /var/containerd
# fstab에 mount 작업
blkid
  ...
  /dev/mapper/pvdata-data: UUID="080e39ac-96dc-490a-82ea-ed7525f6df61" BLOCK_SIZE="4096" TYPE="ext4"
  /dev/mapper/gitdata-data: UUID="fa986196-38ac-440d-b1c8-1d62eea146b6" BLOCK_SIZE="4096" TYPE="ext4"
  ...
vi /etc/fstab (입력)
  ...
  UUID="080e39ac-96dc-490a-82ea-ed7525f6df61" /var/openebs/local  ext4  defaults 0  1
  UUID="fa986196-38ac-440d-b1c8-1d62eea146b6"  /var/containerd    ext4  defaults  0  1 
  ...
mount -a
df -h |grep data
  /dev/mapper/pvdata-data   126G   36G   84G  31% /var/openebs/local
  /dev/mapper/gitdata-data  126G   71G   49G  60% /var/containerd
```

* containerd 설정 변경 (worker node 만 해당)

```bash
cd /etc/containerd
vi config.toml
  ...
  root = "/var/lib/containerd" # gitpod의 경우 새로 생성한 filesystem mountpoint
  ...
systemctl restart containerd
```

#### 2.3.3 install kubernetes cluster with kubekey

* kubekey kubernetes cluster 생성을 위한 config 설정 (master node root 사용자)

```bash
kk create config --with-kubernetes v1.24.9
```

* spec.hosts.\[] 부분을 준비된 vm에 맞게 조정

```yaml
apiVersion: kubekey.kubesphere.io/v1alpha2
kind: Cluster
metadata:
  name: moornmo
spec:
  hosts:
  - {name: master, address: 218.236.22.35, internalAddress: 218.236.22.35,  privateKeyPath: "~/.ssh/id_ed25519"}
  - {name: node1, address: 218.236.22.36, internalAddress: 218.236.22.36,  privateKeyPath: "~/.ssh/id_ed25519"}
  - {name: node2, address: 218.236.22.37, internalAddress: 218.236.22.37,  privateKeyPath: "~/.ssh/id_ed25519"}
  - {name: node3, address: 218.236.22.38, internalAddress: 218.236.22.38,  privateKeyPath: "~/.ssh/id_ed25519"}
  roleGroups:
    etcd:
    - master
    control-plane: 
    - master
    worker:
    - node1
    - node2
    - node3
  controlPlaneEndpoint:
    ## Internal loadbalancer for apiservers 
    # internalLoadbalancer: haproxy

    domain: lb.kubesphere.local
    address: ""
    port: 6443
  kubernetes:
    version: v1.24.9
    clusterName: cluster.local
    autoRenewCerts: true
    containerManager: containerd
  etcd:
    type: kubekey
  network:
    plugin: calico
    kubePodsCIDR: 10.233.64.0/18
    kubeServiceCIDR: 10.233.0.0/18
    ## multus support. https://github.com/k8snetworkplumbingwg/multus-cni
    multusCNI:
      enabled: false
  registry:
    privateRegistry: ""
    namespaceOverride: ""
    registryMirrors: []
    insecureRegistries: []
  addons: []
```

* kubekey로 kubernetes cluster 생성

```bash
# k8s v1.24 부터container runtime은 docker 제외
kk create cluster -f config-sample.yaml --container-manager containerd
kubectl get pod -A
```

* kubernetes 운영을 위한 bash cli 환경 설정

> vi \~/.bashrc

```bash
# ~/.bashrc 하단에 추가
...
alias k=kubectl
source <(kubectl completion bash)
complete -F __start_kubectl k
alias v=velero
source <(velero completion bash)
complete -F __start_velero v
alias kx='f() { [ "$1" ] && kubectl config use-context $1 || kubectl config current-context ; } ; f'
alias kn='f() { [ "$1" ] && kubectl config set-context --current --namespace $1 || kubectl config view --minify | grep namespace | cut -d" " -f6 ; } ; f'
```

* kubesphere 설치

> KubeSphere는 풀 스택 자동화 및 간소화된 DevOps 워크플로를 지원하는 멀티테넌트 엔터프라이즈급 컨테이너 플랫폼입니다.

```bash
$ kubectl apply -f https://github.com/kubesphere/ks-installer/releases/download/v3.3.2/kubesphere-installer.yaml
$ kubectl apply -f https://github.com/kubesphere/ks-installer/releases/download/v3.3.2/cluster-configuration.yaml
$ kubectl logs -n kubesphere-system $(kubectl get pod -n kubesphere-system -l 'app in (ks-install, ks-installer)' -o jsonpath='{.items[0].metadata.name}') -f
kubesphere web browser 접속
http://218.236.22.60:30880
password 변경
```

#### 2.3.4 kubernetes 주요 echo system 설치

* openebs 설치 (CSI)

> 각 Kubernetes 노드에서 사용 가능한 스토리지를 관리하고 해당 스토리지를 사용 하여 Stateful 워크로드에 로컬 또는 분산 영구 볼륨을 제공한다

```bash
# manifests file로 설치
$ kubectl apply -f https://openebs.github.io/charts/openebs-operator-lite.yaml
$ kubectl apply -f https://openebs.github.io/charts/openebs-lite-sc.yaml
# openebs-hostpath 를 default storageClass로 설정
kubectl patch storageclass openebs-hostpath -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
$ kubectl get sc
```

* openebs memory leak 방지를 위해 data.node-disk-manager.config.filterconfigs\[2].exclude 항목에 yaml 내용 추가

> kubectl edit cm openebs-ndm-config -n openebs ()

```yaml
...
"/dev/loop,/dev/loop0,/dev/loop1,/dev/loop2,/dev/loop3,/dev/loop4,/dev/loop5,/dev/loop6,/dev/loop7,/dev/sda1,/dev/fd0,/dev/sr0,/dev/sr1,/dev/ram,/dev/md,/dev/dm-,/dev/rbd,/dev/zd"
...
```

> openebs namespace 내에 openebs-ndm-\* pod를 delete 한다.

```bash
kubectl -n openebs delete pod openebs-ndm-xxxxx
```

* metallb 설치 (https://metallb.universe.tf/)

> Software로 Load Balancer 기능을 구현하여 Bare-metal Kubernetes cluster에서 Network Load Balancer 없이 Load Balancing 기능을 제공 하는 오픈소스 솔루션

```bash
# edit kube-proxy configmap
$ kubectl get configmap kube-proxy -n kube-system -o yaml | sed -e "s/strictARP: false/strictARP: true/" | kubectl apply -f - -n kube-system
# deploy metallb
$ kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.10/config/manifests/metallb-native.yaml
# metallb L2 mode configure (node network와 동일한 network 대역)
$ echo '{"apiVersion":"metallb.io/v1beta1","kind":"L2Advertisement","metadata":{"name":"dev-advert","namespace":"metallb-system"},"spec":{"ipAddressPools":["dev"]}}' | k apply -f -
$ echo '{"apiVersion":"metallb.io/v1beta1","kind":"IPAddressPool","metadata":{"name":"dev","namespace":"metallb-system"},"spec":{"addresses":["172.168.0.100-172.168.0.110"]}}' | k apply -f -
```

* NFS client provisioner

> PersistentVolumeClaim 를 통해 기존에 구성된 NFS 서버에 Persistent Volume 를 자동으로 생성하는 도구

```bash
# nfs server 확인 (node에 os에서 제공하는 nfs-client가 설치되어있다)
$ showmount -e 172.168.x.x # export 된 directory가 표시된다
$ helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
$ helm repo update
$ helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner --set nfs.server=172.168.x.x  --set nfs.path=/nfs
$ kubectl get sc
```

* nginx install (https://www.nginx.com/)

> Kubernetes 및 컨테이너화된 환경의 클라우드 네이티브 앱을 위한 트래픽 관리 솔루션

```bash
$ helm repo add nginx-stable https://helm.nginx.com/stable
$ helm repo update
$ helm install nginx-ingress nginx-stable/nginx-ingress --create-namespace --namespace nginx-ingress
$ kubectl -n nginx-ingress get svc
```

* install cert-manager (https://cert-manager.io/)

> Cert Manager는 K8s 내부에서 https 통신을 하기 위한 인증서를 생성하고 자동으로 갱신해주는 역할을 하는 모듈

```bash
$ helm repo add jetstack https://charts.jetstack.io
$ helm install cert-manager jetstack/cert-manager -n cert-manager --create-namespace --version v1.12.0 --set installCRDs=true
# cert-manager를 사용한 인증서 관리 (DNS서버는 aws route53)
$ aws configure  # handymes.com 계정 설정
# route53 hosted-zone id 획득 및 확인
$ aws route53 get-hosted-zone --id Z0544193SFSS591OE3S6 # 등록된 domain 정보출력
```

> aws access key secret object로 변환

```bash
$ kubectl create secret generic awssecretkey --from-literal secret-access-key=hzLx2HdJq**************/yBT*****n6FgEA
```

> DNS-01 clusterIssuer 생성

```bash
echo 'apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: route53-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: zasmin@uengine.net
    privateKeySecretRef:
      name: aws-secret-prod
    solvers:
    - selector:
        dnsZones:
          - "handymes.com"
      dns01:
        route53:
          region: ap-northeast-2
          hostedZoneID: Z05441************S591OE3S6
          accessKeyID: AKIA4O**********YBEN
          secretAccessKeySecretRef:
            name: awssecretkey
            key: secret-access-key' | kubectl apply -f -
```

> 생성된 clusterissuer 로 certificate 요청

```bash
echo '{"apiVersion":"cert-manager.io/v1","kind":"Certificate","metadata":{"name":"cert-manager-prod-cert","namespace":"cert-manager"},"spec":{"secretName":"aws-prod-tls","issuerRef":{"name":"route53-prod","kind":"ClusterIssuer"},"commonName":"*.handymes.com","dnsNames":["*.handymes.com"]}}' | kubectl apply -f -
```

> 인증서 진행/확인 명령어

```bash
$ kubectl get challenges.acme.cert-manager.io 
$ kubectl get certificates
$ kubectl get certificaterequest
$ kubectl get order
$ kubectl get secret # aws-prod-tls 확인
```

* install kubed

> Kubernetes Operator 로 ConfigMap 과 Secret 을 namespace 간에 동기화해주는 역할을 한다

```bash
$ helm repo add appscode https://charts.appscode.com/stable/
$ helm repo update
$ helm install kubed appscode/kubed --version 0.12.0 --namespace kube-system --wait \
--set apiserver.enabled=false \
--set config.clusterName=cluster.local
```

> 클러스터 전역에서 사용할 Secrets/configMaps에 annotation 을 설정하고 사용할 namespace에 app=kubed를 적용하면 해당 namespace에 복제한다.

```bash
# 제공할 configMap 이나 secret에 annotation 한다.
kubectl annotate secret aws-prod-tls kubed.appscode.com/sync="app=kubed"
# 사용할 namespace에 label 한다.
$ kubectl label ns demo app=kubed
# demo namespace에 aws-prod-tls secret이 복제된다.
```

* velero 설치 (https://velero.io/docs/main/)

> Kubernetes 클러스터 자원과 Persistent Volume의 백업 및 복구 기능을 제공하는 오픈 소스

```bash
$ curl -fsSL -o velero-v1.11.0-linux-amd64.tar.gz https://github.com/vmware-tanzu/velero/releases/download/v1.11.0/velero-v1.11.0-linux-amd64.tar.gz
$ tar -xf velero-v1.11.0-linux-amd64.tar.gz
$ sudo mv velero-v1.11.0-linux-amd64/velero /usr/local/bin
```

> kubernetes에 설치

```bash
# public.crt 는 minio tls public 인증서 ( lets-encrypt fullchain.pem)
# minio-secret는 minio bucket 계정 key 정보
$ velero install --provider aws  --plugins velero/velero-plugin-for-aws:v1.3.0 --bucket mrn-velero --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=https://minio.msafab.com:9000 --cacert public.crt --secret-file ./minio-secret --use-volume-snapshots=false
```

* install istio

> 마이크로서비스 간 데이터 공유를 제어하는 기반을 제공하는 오픈소스 서비스 메쉬 플랫폼입니다. 여기에는 Istio를 모든 로깅 플랫폼, 텔레메트리 또는 정책 시스템으로 통합하도록 지원하는 API가 포함

```bash
$ curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.18.0 TARGET_ARCH=x86_64 sh -
$ curl -L https://istio.io/downloadIstio | sh -
$ cd istio-1.18.0
$ export PATH="$PATH:/home/atid/istio-1.18.0/bin"
$ istioctl x precheck
$ istioctl install
# demo namespace에 istio sidecar 적용예
$ kubectl label namespace demo istio-injection=enabled
```

* install argocd

> k8s 애플리케이션 배포 및 관리를 자동화하는 오픈소스 도구입니다. GitOps 방식을 지원하며, Git 리포지토리에 저장된 애플리케이션 코드와 kubernetes 리소스 정의 파일을 기반으로 배포를 수행

```bash
$ helm repo add argo https://argoproj.github.io/argo-helm
$ helm fetch argo/argo-cd
# argocd-redirect-loop 제거 방법
# values.yaml server.extraArgs 항목에 --insecure 추가 (1501 line 인근)
$ kubectl create ns argocd
$ helm install argocd . -n argocd
```

* install gitlab

> 소프트웨어 개발 및 협업을 위한 올인원 솔루션을 제공하는 웹 기반 DevOps 플랫폼이다

```bash
$ helm repo add gitlab https://charts.gitlab.io/
$ helm repo update
$ kubectl create namespace gitlab
$ kubectl create secret generic gitlab-initial-root-password --from-literal=password="asdgasdfa123" -n gitlab
$ kubectl label namespace gitlab app=kubed # 인증서 복사
$ helm fetch gitlab/gitlab
$ tar xzf gitlab-7.1.2.tgz
$ cd gitlab
$ vi values.yaml # ingress 외
$ vi ./charts/gitlab/charts/gitaly/values.yaml # gitaly storage size 변경
$ helm install gitlab . -n gitlab
$ ingress  annotations 설정 변경 (http 413 error) # 용량 제한
    nginx.ingress.kubernetes.io/proxy-body-size: "0"
    nginx.org/client-max-body-size: "0"
```

\


### 2.4. GidPod 설치

2020년에 출시되었으며, '즉시 개발 가능한' 환경을 자동화하는 오픈소스 플랫폼

* 설치 문서

```
1. https://www.gitpod.io/docs/configure/self-hosted/helm-deprecated/installation/on-kubernetes
2. https://www.gitpod.io/docs/configure/self-hosted/latest/installing-gitpod
```

* 설치 환경 요구사항

```
- linux kernel > 5.4
- ubuntu 20.04 and 22.04
- kubernetes version 1.23.10
- orchestration Platform : k3s
- containerd 1.5 and 1.6
- cni: calico vxlan
- cert-manager > 1.5
- database : mysql
- image repository : minio aws s3
- At least 4 vCPU and 16GB of RAM
- Load Balancer
- k8s cluster (do not use docker)
```

* 사전 준비 작업 (내부 pod를 dns를 통해 호출 )

> coredns 설정 변경

```bash
$ kubectl -n kube-system edit cm coredns # ready 와 kubernetes cluster.local 사이
```

```yaml
apiVersion: v1
data:
  Corefile: |
    .:53 {
        errors
        health {
           lameduck 5s
        }
        ready
        rewrite stop {
           name regex registry.gitpod.handymes.com proxy.gitpod.svc.cluster.local
           answer name prox.gitpod.svc.cluster.local registry.gitpod.handymes.com
        }
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf {
           max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
```

> node label (모든 worker node)

```bash
$ kubectl label node node1 gitpod.io/workload_meta=true gitpod.io/workload_ide=true gitpod.io/workload_workspace_services=true gitpod.io/workload_workspace_regular=true gitpod.io/workload_workspace_headless=true
$ kubectl label node node2 gitpod.io/workload_meta=true gitpod.io/workload_ide=true gitpod.io/workload_workspace_services=true gitpod.io/workload_workspace_regular=true gitpod.io/workload_workspace_headless=true
$ kubectl label node node3 gitpod.io/workload_meta=true gitpod.io/workload_ide=true gitpod.io/workload_workspace_services=true gitpod.io/workload_workspace_regular=true gitpod.io/workload_workspace_headless=true
```

> gitpod 설치도구 kots 설치 (root 사용자)

```bash
sudo -i
curl https://kots.io/install | bash
kubectl kots install gitpod
```

> dns 서버에 아래sodyd A record로 등록 (gitpod proxy loadbalancer IP)

```
gitlab.handymes.com
@.gitpod.handymes.com
*.gitpod.handymes.com
*.ws.gitpod.handymes.co
```

> gitpod 설정 콘솔

```bash
kubectl kots admin-console --namespace gitpod
```
